# Command: scan

Full health check across all 5 dimensions. Read-only — never writes to Mixpanel. Produces a scored report with prioritized findings and auto-offers diagnose or fix.

**Session reads:** `project_id`, `project_name`, `org_id`
**Session writes:** `current_context`, `schema_state`, `context_snapshot_age`, `drift_findings`, `lexicon_score`, `health_score`

---

## Step 1 — Read current context

Read business context for the org and project into `current_context` via `Get-Business-Context`. If no context exists at either level, record that as a finding (Freshness: no context set up) and recommend `prepare-ai-readiness` — but continue the scan for the remaining dimensions.

## Step 2 — Pull live schema

Fetch the current schema into `schema_state`:

- **Events:** `Get-Events` with `include_details: true` — get names, descriptions, tags, verified status, volume signals.
- **Top events by volume:** `Run-Query` (insights, All Events breakdown by Event Name, last 30 days) to rank events by recent activity.
- **Properties:** `List-Properties` for event and user properties.

Timestamp `schema_state`. Partial failure: continue with what loaded, note gaps.

## Step 3 — Check Freshness

Parse the Schema Snapshot section from `current_context` (both org and project level). Look for the timestamp comment (`<!-- VOLATILE — captured [TIMESTAMP] -->`).

- **No Schema Snapshot section:** score 0 — context was written without schema grounding.
- **Timestamp found:** compute age in days. Score: 100 if < 7 days, 80 if 7–14, 60 if 15–30, 30 if 31–60, 0 if > 60.
- **No context at all:** score 0, finding: "No business context configured."

Also check: did the schema change significantly since the snapshot? Compare top 10 events in `schema_state` against events listed in the Schema Snapshot. If > 30% differ, add a finding: "Schema has changed significantly since context was last updated."

## Step 4 — Check Schema coverage

Cross-reference live events against what's mentioned in business context.

**New undocumented events:**
- Events in `schema_state` with meaningful volume (top 50% by 30-day volume) that are NOT mentioned anywhere in `current_context` (org or project) AND have no Lexicon description.
- Each is a finding: "Event `[name]` is active ([volume] in 30 days) but not mentioned in context or described in Lexicon."

**Dead references:**
- Event names mentioned in `current_context` that don't appear in `schema_state` or have zero volume in the last 30 days.
- Each is a finding: "Context references `[name]` but it has zero volume in the last 30 days."

**Score:** `100 - (undocumented_pct × 60) - (dead_ref_count × 10)`, clamped to 0–100.

## Step 5 — Check Completeness (delegate)

If `manage-lexicon` is available, invoke its `score-lexicon` command for this project. Capture the result into `lexicon_score`.

If `manage-lexicon` is unavailable, compute a simplified version: count events and properties with vs without descriptions from the metadata already in `schema_state`. Note in output: "Full Lexicon scoring unavailable — showing simplified coverage."

Use the Lexicon score directly as this dimension's score.

## Step 6 — Check Accuracy

Validate that key definitions in context match live data.

**North star check:** Extract the north star metric from context. Verify the referenced event exists in `schema_state`. If not, finding: "North star references `[event]` which doesn't exist or has no volume."

**Qualified-user check:** Extract the qualified/active user definition. Verify any events it references exist. If not, finding: "Qualified-user definition references `[event]` which doesn't exist."

**Naming convention check:** Extract naming conventions from context (e.g., "Title Case", "snake_case", "[Verified] prefix"). Sample the top 20 events and check if actual naming matches. If < 70% match, finding: "Naming convention in context says `[pattern]` but [X]% of top events don't follow it."

**Score:** Start at 100, deduct 25 per failed check. Clamped to 0–100.

## Step 7 — Check Integrity

If context names specific dashboards or reports (in a "Key Dashboards" or similar section), verify they exist via `Search-Entities`.

- **Dashboard/report found:** pass.
- **Not found:** finding: "Context references dashboard `[name]` which doesn't exist or isn't accessible."

**Score:** `(found / total_referenced) × 100`. If no references in context, score 100 (nothing to break).

## Step 8 — Compute composite score

Apply weights from `references/health-dimensions.md`:

```
health_score = (freshness × 0.15) + (schema × 0.25) + (completeness × 0.25) + (accuracy × 0.25) + (integrity × 0.10)
```

Assign grade: A (90–100), B (75–89), C (60–74), D (40–59), F (0–39).

Sort `drift_findings` by severity: Critical (breaks AI output) > Warning (degrades quality) > Info (cosmetic).

## Step 9 — Output

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  CONTEXT HEALTH — [Project Name]
  Score: [XX]/100 ([Grade])
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Freshness           [XX]/100  (wt 15%)
  Schema coverage     [XX]/100  (wt 25%)
  Lexicon coverage    [XX]/100  (wt 25%)
  Definition accuracy [XX]/100  (wt 25%)
  Reference integrity [XX]/100  (wt 10%)
───────────────────────────────────────────────

  TOP FINDINGS (by impact)
    1. [Critical/Warning/Info] [description] → [skill]
    2. [Critical/Warning/Info] [description] → [skill]
    3. [Critical/Warning/Info] [description] → [skill]
    ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Show up to 10 findings. If more exist, note the count and offer to see all.

## Step 10 — Auto-offer

If findings exist:
```
(a) Diagnose the top finding    (b) Start fixing    (c) Return to menu
```

If no findings (score A):
```
Context is healthy. No action needed.
```

## Audit trail

Write `context-health-runs/[ISO-timestamp]-scan.json`:
```json
{
  "project_id": "...",
  "health_score": 82,
  "grade": "B",
  "finding_count": 4,
  "dimensions": {
    "freshness": 80,
    "schema_coverage": 75,
    "completeness": 88,
    "accuracy": 85,
    "integrity": 100
  }
}
```
