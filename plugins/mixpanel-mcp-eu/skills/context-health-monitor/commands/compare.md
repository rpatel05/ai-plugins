# Command: compare

Show what changed between the current scan and a prior one. Lets the user see whether context health is improving, stable, or degrading over time.

**Session reads:** `health_score`, `drift_findings`, `project_id`
**Session writes:** `last_scan`

---

## Precondition

Requires a current `scan` in this session. If missing, tell the user: "Run `scan` first to get a current baseline." and hand off to scan.

## Step 1 — Find prior scan

Look for the most recent audit trail entry in `context-health-runs/` for this `project_id`. Parse the JSON to load `last_scan`.

If no prior scan exists:
```
No prior scan found for [Project Name].
This scan establishes your baseline — run compare again after your next scan to see changes.
```
Return to menu.

## Step 2 — Compute deltas

Compare current scan results against `last_scan`:

- **Score delta:** current `health_score` minus prior.
- **Per-dimension deltas:** each dimension's score change.
- **New findings:** findings in current scan not present in prior.
- **Resolved findings:** findings in prior scan not present in current.
- **Persistent findings:** findings present in both scans.

## Step 3 — Output

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  CONTEXT HEALTH — [Project Name]
  Compare: [prior date] → [current date]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Overall score     [prior] → [current]  [+/-delta]

  Freshness         [XX] → [XX]  [+/-]
  Schema coverage   [XX] → [XX]  [+/-]
  Lexicon coverage  [XX] → [XX]  [+/-]
  Accuracy          [XX] → [XX]  [+/-]
  Integrity         [XX] → [XX]  [+/-]
───────────────────────────────────────────────

  RESOLVED since last scan
    - [description]
    - [description]

  NEW since last scan
    + [description]
    + [description]

  PERSISTENT (still open)
    ~ [description] (first detected [prior date])

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Step 4 — Interpret

Add a one-line summary based on the trajectory:

- **Improving (delta > +5):** "Context health is improving. [N] findings resolved since [date]."
- **Stable (delta -5 to +5):** "Context health is stable. [N] findings remain open."
- **Degrading (delta < -5):** "Context health is declining — [N] new findings since [date]. Consider running fix."

## Follow-on

```
(a) Diagnose a specific finding    (b) Start fixing    (c) Return to menu
```
