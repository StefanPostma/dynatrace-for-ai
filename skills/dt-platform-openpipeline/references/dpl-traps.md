# DPL patterns that fail silently

`parse` does not error on a non-match — it yields null captures. A wrong pattern therefore looks
exactly like "no data", which makes these traps expensive. All verified against a live tenant.

## `LD` requires at least one character

```
parse log.message, "LD '%ASA-' INT:sev ..."
```

fails whenever `log.message` **starts with** `%ASA-`. `LD?` and `(LD)?` do **not** fix it.

**Fix** — run an anchored pattern and an `LD`-prefixed one, and coalesce:

```
parse x, "'%ASA-' INT:sev_a '-' INT:mid_a ': ' LD:msg_a"
| parse x, "LD '%ASA-' INT:sev_b '-' INT:mid_b ': ' LD:msg_b"
| fieldsAdd severity = coalesce(sev_a, sev_b)
```

## Alternation does not backtrack past a partial match

```
(TIMESTAMP('MMM d HH:mm:ss'):ts | ISO8601:ts | WORD:ts)
```

`2025-10-30T11:04:25.123Z` is *partially* consumed by `ISO8601`, which rejects fractional seconds —
and the whole parse then fails instead of trying the next alternative. Adding a later branch does
not rescue it.

**Fix** — a separate `parse` statement, applied only when the primary failed:

```
parse content, "<primary>"
| fieldsAdd primary_ok = isNotNull(<a field the primary sets>)
| parse content, "<fallback>"
| fieldsAdd host.name = if(primary_ok, host.name, else: host_name_alt)
```

⚠️ **Gate the fallback.** Ungated, a loose fallback matches `Oct`/`30` as timestamp/host on RFC3164
lines and corrupts records the primary parsed correctly. `coalesce` is not enough — the fallback
produces a *non-null wrong* value, so it must be suppressed on success, not merely deprioritised.

## `WORD` excludes dots and slashes

It truncates `10.2.3.4` to `10`, `fw01.corp.example` to `fw01`, and misses `/dev/pts/0` entirely.

**Fix** — use a typed matcher (`IPADDR`) or `LD … SPACE` to run to the next delimiter.

## Optionality binds to a group, not a field

```
IPADDR:host.ip?     <- SYNTAX ERROR: mismatched input '?'
(IPADDR:host.ip)?   <- correct
```

This one at least fails loudly, at deploy time.

## `ISO8601` rejects fractional seconds

`2025-10-30T11:04:25Z` parses; `2025-10-30T11:04:25.123Z` does not. Combine with the no-backtracking
rule above and a single alternation cannot cover both — use a second statement.

## Anchor on a distinguishing literal

`' uid='` and `'auid='` both contain `uid=`. Anchor with the leading space or you capture the wrong
field. Prefer literal anchors (`' [' … '] '`) over positional assumptions.

## Test before you ship

Run the pattern as a query first — `data json: """[{"content":"…"}]""" | parse content, "…"` — which
gives a fast loop against real sample lines. Then deploy: the tenant is the only authority on which
functions are enabled inside a processor.
