# Structuring the semantic layer: YAML vs Markdown

The move from "one YAML blob" to "separate layers" was about *what* goes where. Format — YAML vs Markdown vs hybrid — is a different axis: it's about *how* each layer is physically stored, and granularity (one file per layer vs one file per concept). MD and YAML aren't substitutes for each other — they're good at different halves of what you've been writing.

## What each format is actually good at

| | YAML | Markdown |
|---|---|---|
| Machine-parseable (`yaml.safe_load`, dict access) | ✅ Native | ❌ Needs a parser convention (frontmatter, regex) |
| Enforces structure (every metric has `sql`, every entity has `table`) | ✅ Schema-like by nature | ❌ Nothing stops someone writing a paragraph instead |
| Good for exact things: column names, join keys, SQL formulas, status codes | ✅ | ⚠️ Works but loses the "this is a literal string" guarantee |
| Good for prose: caveats, open questions, "why we defined it this way", examples | ⚠️ Awkward as a long `description:` string | ✅ Natural |
| Works with keyword/substring retrieval (what your notebook does today) | ✅ Iterate dict keys directly | ⚠️ Needs chunking first |
| Works with embedding-based/semantic retrieval | ⚠️ Key-value pairs embed poorly | ✅ Prose embeds well |
| Diff-friendly / reviewable by non-engineers (PM, data steward) | ❌ Syntax-sensitive, easy to break indentation | ✅ Just text |

Notice the split: **everything that has to be exactly right for SQL to run** (table names, column names, join paths, taxonomy codes, metric formulas) belongs in YAML, because a typo there breaks a query silently. **Everything that's judgment, narrative, or still being decided** (an unresolved note, a "confirm with the KPI owner" flag, the reasoning behind why `active_users` uses 2 bookings not 1) belongs in Markdown, because that's context a human needs to read, not a value SQL needs to substitute.

This maps almost exactly onto what you've already been producing: your `sql:` and `column:` fields → YAML. Your `# UNCONFIRMED: ...` comments and the "confirm before formalizing" notes → Markdown. You've effectively been writing both already, just interleaved in one file.

## Recommended structure

```
semantic-layer/
├── ontology/
│   ├── entities.yaml          # tables, columns, types, FKs — hard facts only
│   └── notes.md                # "why is this modeled this way" narrative, per-entity gotchas
├── taxonomy/
│   ├── booking_status.yaml
│   ├── property_type.yaml
│   └── clickstream_event.yaml  # one file per controlled vocabulary — small, stable, rarely conflicts
├── metrics/
│   ├── active_users.md         # hybrid: frontmatter + prose (see below)
│   ├── booking_conversion_rate.md
│   └── ...
└── glossary/
    └── open_questions.md       # every unresolved/unconfirmed item lives here, not as null in YAML
```

Two choices worth calling out:

- **Taxonomy stays pure YAML.** It's small, closed-set, and every value is a literal string your SQL needs verbatim — Markdown adds nothing here.
- **Metrics become hybrid files**, one per metric, not one big `metrics.yaml`. This is the same pattern dbt's semantic layer and LookML use: a small YAML frontmatter block for the machine-consumed formula, a Markdown body for the human-consumed reasoning.

## Worked example: converting `active_users`

Instead of this (what you have now):

```yaml
active_users:
  definition: "Users with 2+ bookings in trailing 12 months"
  sql: "COUNT(DISTINCT user_id) FROM bookings WHERE created_at >= ..."
```

Split it into `metrics/active_users.md`:

```markdown
---
name: active_users
table: bookings
sql: |
  COUNT(DISTINCT user_id) FROM bookings
  WHERE created_at >= (SELECT MAX(created_at) FROM bookings) - INTERVAL 12 MONTHS
  GROUP BY user_id HAVING COUNT(*) >= 2
depends_on: [bookings.user_id, bookings.created_at]
status: confirmed
---

## Definition
Users with 2 or more bookings in the trailing 12 months.

## Decisions
- Threshold is **2 bookings**, not 1 — confirmed by business, not the original
  guess (was initially assumed to include any clickstream activity).
- Anchored to `MAX(created_at)` rather than `current_date()` because this is a
  static sample dataset, not a live feed — using the wall clock would show
  zero active users if the data is stale.

## Related
- `active_properties` mirrors this same 2-booking / 12-month logic.
```

And the amenity gap moves out of a `null`-valued metric into `glossary/open_questions.md`:

```markdown
## Amenity ↔ property relationship (blocks q21, amenity_affinity)
`amenities.amenity_id` does not join directly to `reviews.property_id` — these
are not the same key. A bridge table (likely `property_amenities`) probably
exists but hasn't been confirmed with `DESCRIBE TABLE EXTENDED`. Do not
fabricate this join.

**Owner:** unassigned
**Status:** blocking amenity-affinity KPI
```

## Why this matters for your retriever specifically

Your `retrieve_semantic_context` function currently does keyword/substring matching over dict keys — that works fine against YAML because you're matching literal metric names and taxonomy values. If you keep pure keyword matching, the YAML-heavy parts (taxonomy, entity facts) don't need to change at all.

The Markdown files only pay off once you either (a) start chunking + embedding them for semantic search — so a query like "how many people keep coming back" can match your `active_users` doc even without the literal word "active", or (b) start handing them to an LLM as retrieved context directly, where prose reasoning ("why 2 bookings not 1") actually helps the model write correct SQL in edge cases, not just plausible SQL.

If you're staying with keyword matching for now, you can defer the Markdown split and keep everything in YAML with long `description:` strings — it'll work, just be less pleasant for your colleague to review and edit.