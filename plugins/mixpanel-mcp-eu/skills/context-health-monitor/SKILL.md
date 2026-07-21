---
name: context-health-monitor
license: Apache-2.0
description: >
  Detect when Mixpanel business context or Lexicon metadata has drifted from
  the live schema — before the AI starts giving wrong answers. Use whenever a
  user asks "is my context up to date", "why is the AI giving wrong answers",
  "context health", "what changed since we set up", "check for drift", "is my
  project still AI-ready", "stale context", "context audit", or reports that
  MCP output quality has degraded. Also use proactively after product launches,
  schema changes, or org restructures. Owns detection and diagnosis; delegates
  remediation to prepare-ai-readiness (business context) and manage-lexicon
  (Lexicon metadata). Do NOT use for: initial context setup
  (→ prepare-ai-readiness), Lexicon enrichment (→ manage-lexicon), learning
  MCP (→ mcp-guide), building dashboards, or deleting data. Requires Mixpanel
  MCP. Works best with manage-lexicon and prepare-ai-readiness available.
---

# Context Health Monitor

This skill detects context drift — the silent decay that happens when a Mixpanel project's business context and Lexicon metadata fall out of sync with the live schema. Products evolve, events ship, definitions change, but the context the AI reads stays frozen. Without monitoring, the AI gives confidently wrong answers and no one knows why until a customer complains.

The skill is the observability layer in a three-skill lifecycle:

```
prepare-ai-readiness     →     mcp-guide     →     context-health-monitor
      Setup                      Learn                   Maintain
    "Get ready"              "Use it well"           "Keep it working"
```

It checks five health dimensions (mapped from data observability principles), scores them, and routes fixes to the skills that already know how to write safely.

---

# Critical constraints

1. **Read-only by default.** `scan` and `diagnose` never write to Mixpanel. Only `fix` triggers writes, and only by delegating to another skill with its own preview/confirm gates.
2. **Delegate, don't duplicate.** Lexicon scoring → `manage-lexicon`. Context rewrites → `prepare-ai-readiness`. Never reimplement their write, backup, or confirmation logic.
3. **Evidence-grounded findings.** Every finding must cite the specific event, property, or context section that triggered it. No vague "your context may be outdated."
4. **Severity-ranked output.** Findings ordered by impact on AI output quality, not by raw count. A stale "qualified user" definition hurts the AI more than 3 undescribed low-volume events.
5. **Graceful degradation.** If `manage-lexicon` or `prepare-ai-readiness` aren't available, run what you can and note what was skipped — never block entirely.

---

# Health dimensions

Five checks, mapped from data observability pillars. Full definitions and scoring in `references/health-dimensions.md`.

| Dimension | What it checks | Weight |
|---|---|---|
| **Freshness** | Schema Snapshot age. When was business context last updated? Stale = 30+ days with schema changes. | 15% |
| **Schema coverage** | New events firing that aren't in context or Lexicon. Events referenced in context that no longer exist or have zero volume. | 25% |
| **Completeness** | Lexicon metadata coverage — event/property descriptions, tags, display names. Delegated to `manage-lexicon` `score-lexicon`. | 25% |
| **Accuracy** | Do definitions in context match live data? North star event exists? Qualified-user definition references real events? Naming conventions match actual patterns? | 25% |
| **Integrity** | Dashboards or reports named in context — do they still exist and are they accessible? | 10% |

Composite score: 0–100, graded A–F (consistent with `manage-lexicon`).

---

# Components

## Canonical commands

Each command lives in `commands/` and is loaded on demand.

| Command | File | Match if message contains any of |
|---|---|---|
| `scan` | `commands/scan.md` | scan, health check, check context, is my context up to date, drift, stale, audit context |
| `diagnose` | `commands/diagnose.md` | why, tell me more, explain, what changed about [X], drill into, details |
| `fix` | `commands/fix.md` | fix, update, remediate, refresh, bring up to date, repair |
| `compare` | `commands/compare.md` | compare, diff, what changed since last scan, before and after, trend |

If a message matches more than one, show the Command menu.

## Command menu

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Context Health Monitor — [Project Name]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  1. Scan       — Full health check across all 5 dimensions
  2. Diagnose   — Deep dive into a specific finding
  3. Fix        — Route to the right skill to remediate
  4. Compare    — Diff current state vs last scan
  5. Exit
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Session vocabulary

| Key | Shape | Description |
|---|---|---|
| `project_id`, `project_name` | string | Active project. |
| `org_id`, `org_name` | string | Active organization. |
| `current_context` | map | `{ org, project }` — business context read at scan start. |
| `schema_state` | map | Live schema: top events by volume, properties, integrations. Timestamped. |
| `context_snapshot_age` | string | Parsed age of the Schema Snapshot section in business context. |
| `drift_findings` | array | Detected issues. Each: `{ dimension, severity, description, evidence, remediation_skill, remediation_command }`. |
| `lexicon_score` | map | Coverage scores from `manage-lexicon` score-lexicon. |
| `health_score` | number | Composite 0–100 score. |
| `last_scan` | map | Prior scan results from audit trail, for `compare`. |

## Behaviour rules

1. **No phase narration.** Output findings, scores, recommendations. Not "I'll now check your schema..."
2. **Scan is always the entry point.** `diagnose`, `fix`, and `compare` require a prior scan in session. If the user jumps to one without scanning, run `scan` first.
3. **One finding at a time in diagnose.** Don't dump all findings — let the user pick which to explore.
4. **Fix routes, not rewrites.** `fix` identifies the target skill and hands off. It never writes directly.
5. **Re-scan after fix.** After any remediation completes, offer to re-scan so the user sees the score improve.
6. **Audit trail.** After every scan, write `context-health-runs/[ISO-timestamp]-scan.json` with `project_id`, `health_score`, `finding_count`, per-dimension scores.
7. **`exit` always valid.**

---

# Execution

## 1. Resolve project and org

Identify the Mixpanel project. If ambiguous, ask. Read org context if available. If Mixpanel MCP is unavailable, tell the user to connect it and stop.

## 2. Command loop

For each user request:

### 2a. Choose command

- **Explicit:** user names a command → use it.
- **Implicit:** message matches one trigger phrase → use that command.
- **Ambiguous or none:** show the Command menu.

### 2b. Load command

Read `commands/[command].md` if not already in context.

### 2c. Execute command

Follow the command file's instructions. Reuse session state — never re-fetch what already exists.

### 2d. Complete command

Print `Done.` Write audit entry (for `scan`). Honour follow-on offers (Scan → Diagnose/Fix; Fix → Re-scan). Return to command selection.

---

# Delegation contracts

## → manage-lexicon
- `score-lexicon` during scan for Lexicon coverage scores.
- `enrich-and-tag` during fix for Lexicon gaps.
- `review-issues` during fix for data quality issues.
- Respect its own preview/confirm gates; never bypass.

## → prepare-ai-readiness
- `setup-context` or `import-context` during fix for business context updates.
- Full-replace API constraint: the fix must read → patch → preview → confirm → write.
- Respect its backup-before-write rule.

## → mcp-guide
- Reference only; no programmatic handoff.
- If the root cause is prompting habits rather than stale context, suggest mcp-guide.

---

# Reference files
- `references/health-dimensions.md` — dimension definitions, scoring weights, grade thresholds, evidence examples, severity classification.
