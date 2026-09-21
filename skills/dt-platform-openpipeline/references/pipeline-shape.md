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

## Producing OCSF, not just forwarding to `security.events`

A Data Extractor forwards **Semantic Dictionary** fields. That is enough for Dynatrace's own
`security.events` configuration, but a consumer that expects **OCSF** (Amazon Security Lake, the
tenant's OCSF connection endpoint, any OCSF SIEM) needs a real OCSF document, and the `ocsf_001`
bundle only *parses* OCSF findings — nothing in OpenPipeline *emits* OCSF for you.

Four conventions, verified against the official mapping examples
([github.com/ocsf/examples](https://github.com/ocsf/examples/tree/main/mappings), Windows 4625 →
Authentication 3002). Each is a mistake that produces a document the schema accepts and a consumer
misreads:

1. **`severity_id` must not track the outcome.** The reference maps a *failed* logon to
   `severity_id: 1` / `"Informational"`. Severity is the event's importance; whether it succeeded
   belongs in `status_id` (`0` Unknown, `1` Success, `2` Failure, `99` Other). Deriving severity
   from `event.outcome` promotes every routine auth failure to Medium and makes severity useless
   for triage.
2. **Every `*_id` enum carries a string sibling** — `activity_name`, `category_name`, `class_name`,
   `type_name`, `severity`, `status`, `logon_type`. Consumers that render without the schema loaded
   show bare integers otherwise.
3. **`metadata.profiles` follows the data**, declaring which *optional* field groups the document
   actually carries: `["host"]` when there is a `device`, `["cloud"]` for a cloud API event.
   Hardcoding one advertises objects that are not in the document.
4. **`time` is required** and is Unix **milliseconds**. A record the ingest path never stamped
   serialises as `0`, which reads as 1970 downstream.

`type_uid = class_uid * 100 + activity_id` — derive it, never hand-write it.

Emit an endpoint object (`src_endpoint`, `dst_endpoint`) only when there is an address. A
placeholder like `{"ip": "unknown"}` is worse than the field's absence: it is indistinguishable
from a real observation.

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

