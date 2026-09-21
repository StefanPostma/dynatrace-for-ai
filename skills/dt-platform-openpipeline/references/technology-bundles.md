# Built-in technology bundles

A `technology` processor runs a **pre-built, Dynatrace-maintained parser** for a known source.
Prefer one over hand-written DQL: less to maintain, and consistent Semantic Dictionary output.

```json
{ "id": "syslog_builtin_bundle", "enabled": true, "type": "technology",
  "description": "Built-in Syslog parser",
  "technology": { "technologyId": "syslog_001" } }
```

## `technologyId` is a portable string

It is a **stable, human-readable id** — not a per-tenant UUID — so it can be written by hand and
committed to source control. Examples: `syslog_001`, `aws_cloud_trail_v3`, `servers_nginx_v2`.

It is **validated server-side**. A wrong id is rejected on create with:

```
Value not found for reference 'technology' [processorId: ...]
```

so a successful `create` is proof the id is real. `--dry-run` is local and will not catch a bad one.

A `technology` processor takes **no `matcher`** (unlike `dql`, where one is mandatory). Scope it with
the routing entry instead.

## `allowedConfigurations` — the trap

Each bundle declares which pipeline configuration it may be used in. Most are `logs`; a minority are
`security.events`, and those are **rejected by the logs schema with the same error as a typo**:

```
Value not found for reference 'technology'
```

So a perfectly valid id can look nonexistent purely because it was aimed at the wrong configuration.
Check `allowedConfigurations` before concluding an id is wrong.

`ocsf_001` is the common example — it is a `security.events` bundle, so OCSF cannot be parsed by a
logs pipeline. Reach it either through the security-events ingest endpoint, or by forwarding from a
logs pipeline with a **Data Extractor** (see `pipeline-shape.md`).

## Prefer the non-deprecated id

Many bundles ship an old `_001` beside a current `_v2`/`_v3` (`aws_cloud_trail_001`,
`aws_cloud_trail_v2`, `aws_cloud_trail_v3`). **The deprecated one still validates and still works**,
so nothing warns you — check the catalog and take the current one.

## Listing the catalog

```
GET /platform/openpipeline/v1/technologies
Authorization: Api-Token <token with openpipeline:configurations:read>
```

Returns groups, each with `technologies[]` carrying `id`, `name`, `tags`, `matcher`,
`allowedConfigurations`, `deprecated` and `replacementId`. If your token lacks the scope, read an id
off an existing pipeline in the OpenPipeline UI, or simply try it and let the tenant validate.

Each entry's own `matcher` is worth reading — it tells you what the bundle expects. Two examples with
real consequences:

- **AWS CloudTrail** matches on the **delivery path, not the payload**: it requires Firehose
  attributes (`aws.data_firehose.arn`, `aws.log_group`/`aws.log_stream`) or the S3 route
  (`aws.resource.type`, `dt.da.aws.s3.key.name`). Identical CloudTrail JSON arriving via the
  logs-ingest API, a forwarder or OTLP carries none of those and is **never parsed** — a built-in
  "exists" for the source but not for that route. Point the same bundle at it with your own pipeline
  and a narrow routing matcher.
- **`ocsf_001`** matches `category_uid == 2 AND class_uid == 2002`, i.e. OCSF *findings* only, not
  every OCSF class.

## Before writing custom DQL

1. Check the **Dynatrace Hub** — many extensions ship a ready-made pipeline for their technology
   (view-only; wrap with a pipeline group to add processing, never edit).
2. Check the **bundle catalog** above.
3. Only if neither covers the source — or covers only the envelope and not the fields you need —
   write a `dql` processor, and stack it *after* the bundle rather than replacing it.

Sources with no bundle at time of writing, so custom DQL is justified: Windows Event Log / Sysmon,
Linux auditd, Cisco ASA `%ASA-…` message bodies, Office 365 management activity. Note that
`azure_entra_id_audit_logs_001` is **adjacent but not equivalent** to Office 365 management
activity — same directory, different API and envelope.

## ⚠️ A bundle only fires for the ingest routes its own matcher knows

**Routing a record into a pipeline that contains the bundle is NOT enough.** The bundle carries its
own matcher and that still has to pass. `customMatcher` **narrows** it — it cannot widen it.

Verified on a live tenant, both returning *nothing* for logs-ingest-API records:

| bundle | its matcher requires |
|---|---|
| `syslog_001` | `dt.openpipeline.source == "extension:syslog"` or `log.source == "/var/log/syslog"` |
| `aws_cloud_trail_v3` | Firehose attributes, or S3 (`aws.resource.type` + `dt.da.aws.s3.key.name`) |

So "reuse the bundle" holds only when the data arrives by the route the bundle was built for.

### The fix: shim the matcher, don't reimplement the parser

Add a `fieldsAdd` processor **before** the bundle that sets the attributes its matcher needs. You
then maintain three lines instead of a second parser:

```
fieldsAdd cloud.provider          = coalesce(cloud.provider, "aws")
| fieldsAdd aws.resource.type     = coalesce(aws.resource.type, "AWS::CloudTrail::Trail")
| fieldsAdd dt.da.aws.s3.key.name = coalesce(dt.da.aws.s3.key.name, "non-native-route/cloudtrail.json")
| fieldsAdd cloudtrail.route      = "shimmed"
```

Verified: with this ahead of `aws_cloud_trail_v3`, an ingest-API CloudTrail record is fully parsed
into `audit.action`, `audit.identity`, `audit.result`, `actor.ips`, `aws.arn` and the rest.

Rules:

- **Coalesce-guard everything** so genuine delivery values are never overwritten.
- **Set a marker field** so synthetic values remain auditable.
- **Set them inside the pipeline, not at ingest.** At ingest these attributes change *routing* — a
  record carrying real Firehose/S3 attributes is claimed by the built-in route before your pipeline
  sees it.
- **Disable the shim where the source arrives natively.**

### Expect a different field family than your detections query

A bundle normalizes to *its* model, which may not be the one your detections use.
`aws_cloud_trail_v3` emits the **`audit.*`** model (`audit.action`, `audit.identity`, `audit.result`,
`actor.ips`, `client.ip`), not `event.category` / `event.outcome` / `user_id` / `source.ip`. So the
usual shape is three stages:

```
1. fieldsAdd   shim the bundle's matcher (non-native routes only)
2. technology  the DT-maintained parser
3. dql         map its output onto your field contract — mapping only, never re-parsing
```

