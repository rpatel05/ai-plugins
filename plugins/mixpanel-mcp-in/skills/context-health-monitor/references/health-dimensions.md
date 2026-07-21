# Health Dimensions Reference

Definitions, scoring, and severity classification for the five context health dimensions. Grounded in data observability principles (Freshness, Volume, Schema, Distribution, Lineage) and DAMA-DMBOK data quality dimensions (Completeness, Accuracy, Timeliness, Consistency).

---

## Scoring model

Composite score: weighted average of five dimension scores, each 0–100.

| Dimension | Weight | Rationale |
|---|---|---|
| Freshness | 15% | Stale context is a leading indicator but not always harmful if schema hasn't changed. |
| Schema coverage | 25% | Undocumented events directly cause missing or wrong results. |
| Completeness | 25% | Lexicon gaps force the AI to guess what events and properties mean. |
| Accuracy | 25% | Wrong definitions (north star, qualified user) silently corrupt every analysis. |
| Integrity | 10% | Broken dashboard refs are confusing but don't affect query correctness. |

## Grades

| Score | Grade | Label | Meaning |
|---|---|---|---|
| 90–100 | A | Healthy | Context is current and comprehensive. AI output should be reliable. |
| 75–89 | B | Good | Minor gaps. AI output is mostly reliable; address findings when convenient. |
| 60–74 | C | Drifting | Meaningful drift detected. AI output will degrade on affected areas. Act soon. |
| 40–59 | D | Stale | Significant context decay. AI output is unreliable in multiple areas. Fix now. |
| 0–39 | F | Critical | Context is severely outdated or missing. AI output should not be trusted. |

## Severity classification

Every finding is classified by its impact on AI output quality.

| Severity | Criteria | Examples |
|---|---|---|
| **Critical** | Directly causes wrong AI output on common queries. | North star event doesn't exist. Qualified-user definition references dead event. No business context configured. |
| **Warning** | Degrades AI output quality or causes gaps in coverage. | Active events not in context. Schema Snapshot 30+ days old. Lexicon description coverage below 50%. |
| **Info** | Cosmetic or low-impact. Doesn't materially affect output. | Naming convention mismatch on low-volume events. Dashboard reference not found. Schema Snapshot 15–30 days old with no schema changes. |

## Dimension details

### 1. Freshness

**What it measures:** How recently business context was updated relative to schema changes.

**Inputs:**
- Schema Snapshot timestamp from business context (parsed from `<!-- VOLATILE — captured [TIMESTAMP] -->`)
- Current date
- Schema change detection: do the top 10 events in the snapshot match the current top 10?

**Scoring:**
- < 7 days old: 100
- 7–14 days: 80
- 15–30 days: 60
- 31–60 days: 30
- \> 60 days or no snapshot: 0

**Modifier:** If snapshot is old BUT the top events haven't changed, add +20 (capped at 100). Stale-but-stable is less harmful than stale-and-drifted.

### 2. Schema coverage

**What it measures:** Whether the live schema is reflected in business context and Lexicon.

**Inputs:**
- Live events with 30-day volume (from `schema_state`)
- Events mentioned in business context (text scan of `current_context`)
- Events with Lexicon descriptions

**Checks:**
- **Undocumented events:** Active events (top 50% by volume) not mentioned in context AND without a Lexicon description. Percentage of total active events that are undocumented.
- **Dead references:** Event names in context with zero 30-day volume.

**Scoring:** `100 - (undocumented_pct × 60) - (dead_ref_count × 10)`, clamped 0–100.

### 3. Completeness

**What it measures:** Lexicon metadata coverage — descriptions, tags, display names on events and properties.

**Source:** Delegated to `manage-lexicon` `score-lexicon`. Uses its composite score directly.

**Fallback:** If `manage-lexicon` unavailable, compute a simplified score from metadata already fetched: `(events_with_description / total_events) × 100`.

### 4. Accuracy

**What it measures:** Whether key definitions in business context match the live data.

**Checks (each worth 25 points):**
- **North star:** Does the referenced event exist and have volume?
- **Qualified user:** Do the referenced events exist?
- **Naming conventions:** Do top 20 events follow the described pattern (>70% match)?
- **Customer segments:** Do referenced segment property values still appear in data?

**Scoring:** Start at 100, deduct 25 per failed check. Clamped 0–100. If a check can't run (e.g., no north star in context), skip it and redistribute weight.

### 5. Integrity

**What it measures:** Whether dashboards and reports referenced in business context still exist.

**Inputs:**
- Dashboard/report names extracted from "Key Dashboards" or similar sections in context.
- Existence check via `Search-Entities`.

**Scoring:** `(found / total_referenced) × 100`. If no references in context, score 100 (nothing to break).
