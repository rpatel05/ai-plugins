# Command: fix

Route remediation to the appropriate skill based on the finding type. This command never writes to Mixpanel directly — it hands off to a skill that owns the write safely.

**Session reads:** `drift_findings`, `project_id`, `org_id`
**Session writes:** (none — downstream skills manage their own state)

---

## Precondition

Requires a prior `scan` in this session (`drift_findings` must be populated). If missing, tell the user: "Run `scan` first so I know what needs fixing." and hand off to scan.

## Step 1 — Select findings to fix

If the user named a specific finding, match it. Otherwise, show the prioritized list:

```
Which finding(s) do you want to fix?
  1. [Critical] North star references stale event
  2. [Warning] 4 active events not in context
  3. [Warning] Lexicon description coverage at 45%
  all — fix top 3 in sequence
```

## Step 2 — Route to the right skill

### Routing table

| Finding dimension | Skill | Command | Why |
|---|---|---|---|
| Freshness — stale snapshot | `prepare-ai-readiness` | `setup-context` | Context needs a schema refresh; setup-context pulls fresh schema facts. |
| Freshness — no context | `prepare-ai-readiness` | `import-context` or `setup-context` | Context doesn't exist yet; full setup needed. |
| Schema — undocumented events (no Lexicon entry) | `manage-lexicon` | `enrich-and-tag` | Events need descriptions and tags in Lexicon. |
| Schema — undocumented events (not in business context) | `prepare-ai-readiness` | `setup-context` | Business context needs updating to reference new features. |
| Schema — dead references in context | `prepare-ai-readiness` | `setup-context` | Stale event references need removal from context. |
| Completeness — Lexicon gaps | `manage-lexicon` | `enrich-and-tag` | Descriptions, tags, display names need filling. |
| Completeness — data quality issues | `manage-lexicon` | `review-issues` | Type drift, anomalies need triage. |
| Accuracy — stale definitions | `prepare-ai-readiness` | `setup-context` | North star, qualified-user, or convention definitions need updating. |
| Integrity — dead dashboard/report refs | `prepare-ai-readiness` | `setup-context` | Key Dashboards section needs updating. |

## Step 3 — Check skill availability

Before handing off, verify the target skill is available. If not:

```
This finding needs [skill name] to fix, but it isn't available in this session.
You can:
  (a) Install it from the Mixpanel skills repository
  (b) Fix it manually in the Mixpanel UI — [specific guidance]
  (c) Skip and move to the next finding
```

## Step 4 — Hand off with context

Pass the relevant context to the target skill so it doesn't re-discover what we already know:

- For `prepare-ai-readiness`: surface the specific sections that need updating and why, so the interview or import can focus on gaps rather than starting from scratch.
- For `manage-lexicon`: surface the specific events or properties that need attention, so enrichment can target them.

## Step 5 — After fix completes

When the downstream skill finishes:

```
Fix applied via [skill name].

(a) Re-scan to verify improvement    (b) Fix another finding    (c) Return to menu
```

If the user chooses re-scan, run `scan` again and show the score delta:

```
Score: 62/100 (C) → 78/100 (B)  [+16]
  Schema coverage:  60 → 85  [+25]
  Completeness:     45 → 70  [+25]
  [N] findings resolved, [M] remaining.
```

## Fixing multiple findings

If the user chose "all", process findings in priority order (Critical → Warning → Info). After each fix:
- Re-check if the next finding is still valid (a context rewrite may resolve multiple findings).
- Skip findings that were resolved as a side effect.
- Show running progress: "[1/3] Fixed. [2/3] Next: ..."
