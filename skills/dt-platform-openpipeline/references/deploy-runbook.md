# Deploying a pipeline safely

Examples use [`dtctl`](https://github.com/dynatrace-oss/dtctl); the same sequence applies through the
Settings API.

## 0. Confirm context

```bash
dtctl config current-context
dtctl auth whoami
```

## 1. Create the pipeline

```bash
dtctl create settings -f pipeline.json \
  --schema builtin:openpipeline.logs.pipelines --scope environment --dry-run
dtctl create settings -f pipeline.json \
  --schema builtin:openpipeline.logs.pipelines --scope environment
```

`--dry-run` is **local only**. It validates nothing server-side — bad `technologyId`s, disabled
functions and malformed stages surface only on the real create. Read the error body: it names the
exact `path` and `message` per violation, which is the fastest way to discover an undocumented shape.

Note the returned `objectId`; the pipeline UUID that routing needs is embedded in it:

```bash
python3 -c "import base64,re,sys; \
print(re.search(r'[0-9a-f-]{36}', base64.urlsafe_b64decode(sys.argv[1]+'==').decode('utf-8','replace')).group(0))" \
  "<objectId>"
```

## 2. Append the routing entry — never replace

Routing is a single object holding every team's routes.

```bash
dtctl get settings --schema builtin:openpipeline.logs.routing \
  --scope environment -o json --plain > routing.json
# edit routing.json: APPEND one entry, keep all existing ones, mind the order
dtctl apply -f routing.json --dry-run
dtctl apply -f routing.json
```

Read it back and confirm both your entry **and** everyone else's are still present.

## 3. Updating a live pipeline — fetch and merge

Applying a repo file that contains only `processing` **deletes every other stage**, including a Data
Extractor a colleague added in the UI. Merge instead:

```python
import json, pathlib, subprocess
objs = json.loads(subprocess.run(
    ["dtctl","get","settings","--schema","builtin:openpipeline.logs.pipelines",
     "--scope","environment","-o","json","--plain"],
    capture_output=True, text=True).stdout)

new = json.loads(pathlib.Path("pipeline.json").read_text())
for o in objs:
    if o["value"].get("customId") == new["customId"]:
        o["value"]["displayName"] = new["displayName"]
        o["value"]["processing"]  = new["processing"]   # only this stage
        json.dump([o], open("update.json","w"), indent=2)
```

```bash
dtctl apply -f update.json
```

## 4. Verify with real data

```dql
fetch logs, from: now() - 1h
| filter <your matcher>
| fields timestamp, <extracted fields>, dt.openpipeline.pipelines
| limit 25
```

Diagnosis order:

| symptom | cause |
|---|---|
| `dt.openpipeline.pipelines` = `logs:default` | **routing** never selected the record — fix the routing matcher first |
| pipeline correct, all fields null | processor `matcher` did not match |
| some fields null | the parse pattern — see `dpl-traps.md` |

## Rollback

```bash
dtctl history settings <objectId>
dtctl restore settings <objectId> --version <n>
```

For routing, keep the pre-change `routing.json` and re-apply it. Take that backup **before** the
first change, every time — it is the only cheap undo for a singleton.

## Testing a script without deploying

Processor scripts are DQL, so iterate as a query against synthetic records:

```dql
data json: """[{"content":"<a real sample line>"}]"""
| <the processor script>
| fields <what you expect>
```

This catches pattern bugs in seconds. It does **not** catch functions disabled inside processors
(`in()`, `contains()` and `matchesRegex()` all work here but are rejected there) — only a real
create does.

## ⚠️ Config changes take ~1–2 minutes to reach the ingest path

After `apply`, records ingested immediately still flow through the **previous** pipeline version. A
verification query run straight after deploy therefore returns a **false negative**: the fields look
missing, and the obvious conclusion — "my processor is broken" — is wrong.

Observed live: a corrected processor produced nothing on an ingest ~10s after apply, and worked on an
identical ingest ~2 minutes later with no config change in between. Wait, re-ingest, then judge.

