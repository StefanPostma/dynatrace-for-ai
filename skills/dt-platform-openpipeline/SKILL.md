---
name: dt-platform-openpipeline
description: "Author, validate and deploy Dynatrace OpenPipeline pipelines — built-in technology bundles, DQL processors, matchers, Data Extractors, routing and bucket assignment. Use when building or debugging an ingest-time pipeline: parsing a log source into Semantic Dictionary fields, reusing a built-in bundle, stacking custom DQL on top of one, forwarding logs into security.events, or shipping a pipeline with dtctl. Trigger: \"OpenPipeline processor\", \"technology bundle\", \"technologyId\", \"parse this log at ingest\", \"log pipeline\", \"routing entry\", \"Data Extractor\", \"bucket assignment\". Covers the CONFIGURATION SURFACE — which processor types exist, what each stage accepts, what the API rejects and why. For the DQL subset a processor may use, see dt-obs-log-semantic-mapping. For writing query DQL against Grail, use dt-dql-essentials. Do NOT use for proposing Semantic Dictionary field mappings — that is dt-obs-log-semantic-mapping (logs) or dt-sec-semantic-mapping (security)."
license: Apache-2.0
---

# dt-platform-openpipeline

Build pipelines that parse and enrich records **at ingest**, before Grail storage.

This skill owns the **configuration surface**: what a pipeline object looks like, which processor
types each stage accepts, how built-in bundles are referenced, and which shapes the settings API
rejects. It deliberately does **not** restate the DQL subset a processor may use — that is
[`dt-obs-log-semantic-mapping/references/openpipeline-constraints.md`](../dt-obs-log-semantic-mapping/references/openpipeline-constraints.md)
and the product docs it cites.

## Golden rules

1. **Reuse before you write.** If a built-in technology bundle covers the source, use it and write
   custom DQL only for what it leaves unparsed. See [`technology-bundles.md`](references/technology-bundles.md).
2. **Routing is a destructive singleton.** `builtin:openpipeline.logs.routing` is ONE object holding
   the whole `routingEntries[]` table. A blind `create`/`apply` **replaces the entire table** and
   deletes other teams' routes. Always `get` → append → `apply`.
3. **A pipeline is a multi-stage object.** Applying a file that contains only `processing` silently
   drops a Data Extractor or any other stage someone added in the UI. Fetch and merge instead.
4. **Only a tenant can validate.** `--dry-run` is local. Disabled functions, bad `technologyId`s and
   malformed stages are found only by an actual `create`/`apply`.
5. **Verify empirically.** After deploy, ingest a sample and read it back. A processor that silently
   no-ops is worse than none.

## The processing model

Records flow through fixed **stages**, each an ordered list of **processors**. Routing decides which
pipeline a record enters — **first-match-wins, top to bottom**, so a record enters exactly **one**
pipeline. Stacking therefore happens *within* a pipeline, never by chaining pipelines.

Which processor types each stage accepts (logs configuration, from
`GET /platform/openpipeline/v1/configurations`):

| stage | accepted types |
|---|---|
| `processing` | `technology`, `dql`, `fieldsAdd`, `fieldsRename`, `fieldsRemove`, `drop`, `geoLookup`, `inlineLookup` |
| `dataExtraction` | `securityEvent`, `bizevent`, `sdlcEvent` |

Other configurations (`events`, `security.events`, `bizevents`, `spans`) have their own matrix —
query that endpoint rather than assuming the logs list applies.

## The preferred shape: bundle first, custom DQL after

```
1. type=technology   syslog_001          <- generic envelope, OOTB, no matcher
2. type=dql          vendor header       <- custom
3. type=dql          per-message family  <- custom, gated on what stage 2 produced
```

Make custom stages read `coalesce(log.message, content)` rather than assuming a field name from the
bundle, so the stack survives a change in the bundle's output shape.

## Matchers

- **Every `dql` processor requires a `matcher`** (`Must not be null`). A `technology` processor
  requires none.
- `true` is a valid matcher — the explicit "always" form.
- Available: `matchesValue()` (supports `*`), `matchesPhrase()` (case-insensitive), `isNull()`,
  `isNotNull()`, `iAny()`, `AND`/`OR`/`NOT`, numeric comparators.
- **`in()` and `contains()` are NOT enabled in matchers.** Rewrite set membership as an `or` chain.
- `==` is case-sensitive with no wildcards — use `matchesValue()` when casing can vary.
- **A quoted literal never matches a numeric field.** `result.code == "0"` is accepted by the API,
  deploys cleanly, and is **always false** when `result.code` is a `long` — no error anywhere, the
  processor simply never fires. Comparison does not coerce across types. Drop the quotes for a
  numeric field, or match a sibling string field if the source has one. This is the single most
  expensive silent failure in this skill: every symptom points at the parse pattern, and the parse
  pattern is fine.

  ```
  matcher: "result.code == \"0\""    # long field  -> never fires, silently
  matcher: "result.code == 0"          # long field  -> correct
  matcher: "audit.result == \"success\""  # string field -> correct
  ```

  Check a field's actual type before writing a matcher against it:
  ```dql
  fetch logs, from: now() - 1h | filter isNotNull(result.code) | fieldsAdd t = type(result.code) | fields t | limit 1
  ```

## Quick reference

| Task | Where |
|---|---|
| Find a bundle, read a `technologyId`, check `allowedConfigurations` | [`technology-bundles.md`](references/technology-bundles.md) |
| Pipeline/routing JSON, Data Extractors, stage keys, emitting OCSF | [`pipeline-shape.md`](references/pipeline-shape.md) |
| DPL patterns that silently fail | [`dpl-traps.md`](references/dpl-traps.md) |
| Safe deploy, rollback, stage-preserving update | [`deploy-runbook.md`](references/deploy-runbook.md) |

## Verify

```dql
fetch logs, from: now() - 1h
| filter <your matcher>
| fields timestamp, <the fields you extracted>, dt.openpipeline.pipelines
| limit 25
```

`dt.openpipeline.pipelines` showing `logs:default` means **routing** never selected the record — fix
the routing matcher before suspecting the parse pattern.
