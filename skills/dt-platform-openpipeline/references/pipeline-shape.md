# Pipeline and routing object shapes

## A pipeline carries every stage, not just `processing`

`builtin:openpipeline.logs.pipelines`, one object per pipeline:

```
value
├── customId, displayName, metadataList
├── processing                { processors: [...] }
├── smartscapeNodeExtraction  { processors: [...] }
├── smartscapeEdgeExtraction  { processors: [...] }
├── dataExtraction            { processors: [...] }   <- "Data Extractor" in the UI
├── davis                     { processors: [...] }
├── metricExtraction          { processors: [...] }
├── securityContext           { processors: [...] }
├── security                  { processors: [...] }
├── costAllocation            { processors: [...] }
├── productAllocation         { processors: [...] }
└── storage                   { processors: [...] }
```

⚠️ **`apply` replaces the whole object.** A file containing only `processing` will silently delete a
Data Extractor, a bucket assignment, or anything else a colleague added in the UI. To change one
stage of a live pipeline, fetch it and merge — see `deploy-runbook.md`.

Minimal valid pipeline:

```json
{
  "customId": "pipeline_my_source",
  "displayName": "My source",
  "processing": {
    "processors": [
      { "id": "parse_my_source", "enabled": true, "type": "dql",
        "description": "…",
        "matcher": "matchesValue(log.source, \"my-source\")",
        "dql": { "script": "parse content, \"…\"\n| fieldsAdd event.category = \"authentication\"" } }
    ]
  }
}
```

Processor `id` must be **4–100 characters** (`Size must be between 4 and 100`).

## Routing — the destructive singleton

`builtin:openpipeline.logs.routing` is **one object** holding the entire table:

```json
{ "routingEntries": [
  { "pipelineType": "custom",
    "pipelineId": "<pipeline UUID>",
    "builtinPipelineId": "00000000-0000-0000-0000-000000000000",
    "matcher": "log.source == \"my-source\"",
    "description": "…",
    "enabled": true }
] }
```

- **First-match-wins, top to bottom** — order matters. A vendor-specific pipeline must be listed
  *before* a generic one that would also match, or the generic one wins and the vendor fields never
  appear.
- Never `create` routing — always `get` → append your entry → `apply`.
- `pipelineId` accepts the pipeline's UUID; the server normalises it to the full object reference on
  write. The UUID is embedded in the settings `objectId` (base64-decode it) or visible in the UI.
- Keep matchers **narrow**. A broad custom entry placed above a built-in route will shadow it.

## Data Extractors (`dataExtraction`)

Emit a *new* record into another configuration — `securityEvent`, `bizevent`, or `sdlcEvent`. This is
how a log reaches the `security.events` configuration, and therefore the `security.events`-only
technology bundles that the logs schema rejects.

```json
"dataExtraction": {
  "processors": [
    { "id": "extract_security_event", "enabled": true, "type": "securityEvent",
      "description": "", "matcher": "winlog.eventid == 4625",
      "securityEvent": {
        "fieldExtraction": {
          "type": "include",
          "include": [
            { "extractionType": "field",    "sourceFieldName": "user_id",
              "destinationFieldName": "user_id" },
            { "extractionType": "constant", "constantFieldName": "event.provider",
              "constantValue": "windows.eventlog" }
          ]
        }
      } }
  ]
}
```

| rule | detail |
|---|---|
| `fieldExtraction.type` | `include`, `exclude`, or `includeAll` |
| `include` / `exclude` | **minimum 1 entry** — `[]` is rejected: *"fell below the collection's lower size limit which was set to 1"* |
| entries | **objects**, not strings — a bare `"user_id"` fails with `Must be of type object` |
| `extractionType` | `constant` or `field` |
| `field` | needs `sourceFieldName`; `destinationFieldName` optional |
| `constant` | needs **both** `constantFieldName` and `constantValue` |
| `includeAll` | takes no list |

⚠️ The UI can emit `{"type": "include", "include": []}` for an extractor that has been added but not
yet filled in. That config is **rejected by the settings API**, so a pipeline exported mid-edit will
not re-apply. Complete the list or switch to `includeAll`.

⚠️ **`matcher: "true"` on a Data Extractor forwards every record in the pipeline**, with the volume
and licensing consequences that implies. Scope it unless you really mean everything.

## ⚠️ Never overwrite `timestamp` from the payload

A record whose `timestamp` falls outside the ingest window (roughly 24h) is **silently dropped** —
the ingest API returns **HTTP 204 with no warning** and the record simply never exists.

So a processor that parses an event time out of the log line and assigns it to `timestamp`
**destroys** any record carrying an older time, *after* ingest already accepted it. The failure is
invisible from both ends: the sender sees success, and the record is not in Grail to be missed.

**Verified on a live tenant.** Two identical auditd records, differing only in the epoch embedded in
the line:

| embedded time | `fieldsAdd timestamp = <parsed>` | parsed time in a namespaced field |
|---|---|---|
| now | landed | landed |
| 13 months ago | **gone — HTTP 204, never queryable** | landed, event time preserved |

**Write the parsed event time to a namespaced field** (`auditd.timestamp`, `o365.timestamp`) and
leave the ingest-assigned `timestamp` authoritative:

```
| fieldsAdd auditd.timestamp = coalesce(auditd_ts, timestampFromUnixNanos(toLong(epoch * 1000000000)))
```

This matters most for **replay, backfill and catch-up reads**: a replay harness deliberately rebases
timestamps into the ingest window, and a processor like this silently undoes that and deletes the
data. If the source is always fresh and event-time semantics are wanted, assigning `timestamp` is
defensible — but it is a trade, not a default.

## Other limits the API enforces

| field | limit |
|---|---|
| processor `id` | 4–100 characters |
| processor `description` | **≤ 512 characters** (`Size must be lower than or equal to 512`) |

