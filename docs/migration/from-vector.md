# Migrating from Vector

The agent can take over the **log collection** of a host that runs Vector and keep delivering to the same place:
Grafana Loki, Splunk (HTTP Event Collector), Elasticsearch or OpenSearch, Kafka, Amazon S3 or an S3-compatible
store, or a syslog receiver. Or it can send the same data as RFC 5424 syslog to logrok or any syslog receiver.

You don't rewrite the configuration by hand: the agent reads Vector's configuration — YAML, TOML or JSON, one
file or a whole configuration directory — follows the `inputs` of every transform and sink, and writes an agent
configuration plus a report of everything it could not carry over.

## Is it a fit?

**Verified against:** Vector 0.59.0 (reference documentation read 2026-10-16; the site states no version). A configuration was validated and run by Vector 0.59.0, delivering into Grafana Loki.

The agent is a **log forwarder**. It does not collect metrics or traces, and it has no general-purpose
transformation language. If your Vector reads log files, the journal, syslog or the Windows event log and ships
them with light processing, the conversion is close to a drop-in. If it runs substantial VRL programs, the
programs are listed for you to port, and the rest still converts.

| If your Vector… | Then |
|---|---|
| reads files, the systemd journal, syslog, raw TCP / UDP / Unix sockets, the Windows event log, Kubernetes pod logs, or receives HTTP or Splunk HEC | converted (tables below) |
| uses `filter` and `route` transforms with simple conditions | converted, **per sink**: a filter in front of one sink filters only that sink |
| uses `remap` with plain assignments, `del()` and renames | converted |
| uses `remap` with other VRL (parsing, functions, conditionals), `reduce`, `lua`, `aggregate`, `log_to_metric` | **not converted**: each program is listed with its file and line |
| uses `sample`, `throttle`, `dedupe` | converted to the agent's reduction processors (with stated differences) |
| collects **metrics** (`host_metrics`, `internal_metrics`, `prometheus_*`, …) | not converted, and not carried by the agent; keep Vector for them |
| sends to `loki`, `splunk_hec_logs`, `elasticsearch`, `kafka`, `aws_s3`, or a `socket` sink with the syslog codec | converted |
| sends to `http`, `datadog_logs`, `vector` or other sinks | not converted natively; use `-target syslog` or configure the destination by hand |

## How it fits together

```
before:   files / journal / sockets ──► Vector ──► loki | splunk_hec_logs | elasticsearch | kafka | aws_s3 | socket   (per sink)
after:    files / journal / sockets ──► agent  ──► the same destinations, selected by a condition per output
          (or)                          agent  ──► RFC 5424 syslog over TLS (logrok / any syslog receiver)
```

## Step 1 — convert

Get `luna-migrate` first: a separate download published with every agent release (`luna-migrate-<os>-<arch>`, saved as `luna-migrate` / `luna-migrate.exe`) — see [Getting luna-migrate](../USER-GUIDE.md#getting-luna-migrate).

Run the converter where Vector's configuration is (it only reads files):

```bash
luna-migrate vector /etc/vector/vector.yaml \
  -out /etc/logrok-universal-agent/agent.yaml \
  -report vector-conversion.json
```

```powershell
& ".\luna-migrate.exe" vector "C:\Program Files\Vector\config\vector.yaml" `
  -out "C:\ProgramData\logrok-universal-agent\agent.yaml" `
  -report vector-conversion.json
```

| Flag | Default | Meaning |
|---|---|---|
| `-out` | *(required)* | where to write the agent configuration; an existing file is never overwritten |
| `-report` | — | also write the full report as JSON |
| `-target` | `native` | `native`: send where Vector sent. `syslog`: everything to one RFC 5424 receiver over TLS |
| `-syslog-endpoint` | — | `host:port` of the syslog receiver for `-target syslog` |
| `-syslog-plain` | off | send the syslog output over plain TCP instead of TLS (a receiver without TLS, or a lab) |
| `-start-at` | `end` | where file inputs start on first run: `end` (only new lines: nothing Vector already shipped is shipped again) or `beginning` |

**Formats and several files.** The format comes from the file extension, as in Vector: `.toml`, `.json`, and
`.yaml` / `.yml` (anything else is read as YAML). Vector started with several `--config` files or a glob merges
them; pass the same files to `luna-migrate`, comma separated and in the same order (`a.yaml,b.toml`), or the same
glob. Pass a directory for a `--config-dir` layout: its `.toml`, `.yaml`, `.yml` and `.json` files are read in
name order, and the `sources/`, `transforms/` and `sinks/` sub-directories are read with each file name as the
component id. Files are merged table by table; when two files set the same key, the first one read wins (as in
Vector) and the report lists every such overlap.

**TOML.** `luna-migrate` reads the TOML that Vector configurations use: tables and arrays of tables, dotted and
quoted keys, inline tables, arrays, every string form (basic, literal, multi-line), integers, floats and booleans.
Date and time values are not read; a file holding one is refused with the line number rather than guessed at.

**Environment variables and secrets.** The agent does not expand `${NAME}` or `$NAME` (nor does `luna-migrate`:
the converter never reads Vector's environment), and it does not follow `SECRET[…]` references. An address, a path
or a credential written that way becomes `REPLACE_WITH_…` and is listed under "fill in before deploying"; the
report names the reference.

luna-migrate checks the configuration's structure, then runs the installed agent's `-check-config` (found beside `luna-migrate` or on the `PATH`, or named with `-agent`), which builds every module exactly as that agent version would; a configuration the agent rejects is removed. Without an agent to hand it says so — then run `logrok-universal-agent -check-config -config agent.yaml` on the target host.

## Step 2 — read the report

| Severity | Meaning |
|---|---|
| `manual` | **not carried over**: a person must decide (each line says what to do instead) |
| `warning` | carried over, with a difference to check |
| `info` | carried over differently, nothing to do |
| `skipped` | deliberately not converted: metrics sources, `console` / `blackhole` / `file` sinks, components no sink consumes, Vector's own API and unit tests |

Each finding names the component and setting it came from (`sources.app`, `transforms.parse.source`,
`sinks.loki.labels`, …) with the file and line. For a VRL program that did not convert, the line is the line of
the statement that stopped the conversion. Then fill in what the report lists under "fill in before deploying".
Tokens, passwords, API keys and access keys are **never copied** out of the Vector configuration: the generated
file says `REPLACE_WITH_…` (search for it).

## Topology: which sink receives what

In Vector every transform and sink lists its `inputs` (component ids, `route.branch` outputs, `*` wildcards), so a
transform affects only the sinks downstream of it. The agent has one processing chain and routes with a condition
per output. The converter follows every path from a source to a sink and keeps that structure:

- A **filter**, or a **route** branch, that every destination of a source sits behind becomes a drop rule for that
  source's events — what Vector does to them.
- A filter or route branch that only **some** sinks sit behind becomes part of those outputs' `when:` conditions.
  The other outputs still receive every event. A sink fed by every branch of a route (for example `r.a` and
  `r._unmatched`) needs no condition at all.
- A **remap**, **sample**, **throttle** or **dedupe** applies to the sources that feed it. It must sit in front of
  every sink of those sources: one processing chain cannot modify an event for one output and not for another. When
  it does not, it is **not applied** (`manual`) — the sinks behind it receive the events unmodified, and nothing is
  changed for the other sinks.
- When the converter needs to tell sources apart, each input stamps the field `vector.source` (the Vector source
  id), and the generated rules and conditions are written over that field. When an output needs a `when:`, the
  field is delivered with every event (the condition is evaluated after the processors); otherwise a last rule
  removes it. The report says which case applies. Routing conditions and more than one output are Apex features.
- A sink that Vector fed the same event along two paths received it twice; the agent delivers it once.
- A source whose only sinks are not converted (or are `console` / `blackhole`) is not converted.

## What is converted

### Sources

| Vector | Agent |
|---|---|
| `file` `include`, `exclude`, `encoding.charset` (`utf-8`, `utf-16le`, `utf-16be`), `multiline` | `filetail` `paths` (`path` when `include` lists a single path), `exclude`, `encoding`, `multiline`. `halt_before` becomes `line_start_pattern` (the condition pattern), `halt_with` becomes `line_end_pattern`, `continue_through` becomes `line_start_pattern` (the start pattern; warning: a line matching neither pattern now joins the record), `timeout_ms` becomes `flush_timeout`. The agent keeps its own resume state |
| `journald` `include_units`, `exclude_units` (a name without a dot gets `.service`, as in Vector), `include_matches` on `_SYSTEMD_UNIT` and `SYSLOG_IDENTIFIER`, `exclude_matches` on `_SYSTEMD_UNIT` | `journald` `units`, `identifiers`; excluded units become a drop rule on the `unit` field. An `exclude_matches` on `SYSLOG_IDENTIFIER` (or any other field) is not converted: it is reported as manual and those entries are forwarded. The agent keeps its own journal cursor |
| `windows_event_log` `channels` (comma-separated entries are split), `event_query`, `only_event_ids` / `ignore_event_ids`, `render_message: false` | `windows_eventlog` `channels`, `query`, a structured query per channel for the event-id lists, `render: xml` |
| `syslog` `mode` tcp / udp / unix, `address`, `path`, `tls` (certificate, key, client CA) | `syslog_in` `listen` + `protocols`, `socket` (unix stream), `tls_listen` with the certificate, key and `tls_client_ca_file` |
| `socket` tcp / udp / `unix_datagram` / `unix_stream`, `decoding.codec` `bytes`, `syslog` or `json`, `framing` `newline_delimited` or `octet_counting` | `syslog_in` (`listen`, `socket`, `framing`); the JSON codec becomes a `parse_json` processor. A line that looks like syslog is parsed as syslog |
| `http_server` `address`, `path`, `auth.strategy: bearer`, `tls`, `decoding.codec` `bytes` or `json` | `http_in` (one event per line of the body; `auth_token` placeholder); JSON becomes a `parse_json` processor |
| `splunk_hec` `address`, `valid_tokens`, `tls` | `hec_in` with one token placeholder per token. The agent's receiver **requires a token**: a Vector source without tokens gets a placeholder |
| `kubernetes_logs` | `filetail` on the kubelet's pod log files in container-log mode (`format: auto`, Apex): the namespace, pod, container and container id come from the file path as `k8s_namespace`, `k8s_pod`, `k8s_container`, `container_id`. Labels, annotations and other metadata that Vector read from the Kubernetes API are not added |

### Transforms

| Vector | Agent |
|---|---|
| `filter` (a VRL condition, or `type: vrl`) | a drop rule or output `when:` conditions (see Topology) |
| `route` branches and `_unmatched` | output `when:` conditions or drop rules |
| `remap` made only of `.field = "literal"` (also numbers and booleans, stored as text), `del(.field)` and `.new = del(.old)` | `expr` rules in the same order |
| `sample` `rate` (1 in N) or `ratio`, `key_field`, `exclude` | `sample` (Apex) `ratio`, `key`, `keep_if` |
| `throttle` `threshold`, `window_secs`, `key_field` (`{{ field }}`), `exclude` | `throttle` (Apex) `max`, `window`, `key`, `keep_if`. Vector's limiter refills continuously; the agent's window is fixed, so a burst at a window boundary can briefly pass up to twice the limit |
| `dedupe` `fields.match`, `cache.num_events`, `time_settings.max_age_ms` | `dedup` (Apex) `key`, `max_keys`, `window`. The timestamp is never part of the agent's key; without `max_age_ms` the agent's 60-second window applies |

**Conditions.** A condition converts when its meaning is the same on the agent: comparisons of a field with a
string or number (`==`, `!=`, `<`, `<=`, `>`, `>=`), `&&`, `||`, `!`, parentheses, and the functions `exists`,
`contains`, `starts_with`, `ends_with`, `match` (with an RE2-compatible pattern) and `includes`, with `string!()` /
`to_string!()` around a field. `.message` and `.host` are the event's message and host; other fields are the
event's fields. Everything else — other functions, `case_sensitive: false`, `??`, variables, metadata (`%…`),
array indexing, `null`, `.timestamp`, `.source_type`, negative numbers — is reported with the reason, and the
filter **drops nothing** on the agent (the destinations behind it receive more than before). Two caveats are
reported where they apply: `exists(.f)` reads a field holding an empty value as absent, and `==` with a number
compares numerically when the field holds a number (agent field values are text).

**Field names.** Vector's sources name some fields differently from the agent's inputs: a file's path is the
agent's event source (`file:<path>`) rather than a `file` field; syslog's app name, severity and facility are
header fields of the agent's event; Windows events carry `provider` rather than `provider_name`; the HEC receiver
stores `hec.sourcetype` / `hec.index`. Conditions on such fields need a look; the report says so.

### Sinks

| Vector | Agent |
|---|---|
| `socket` tcp / udp with `encoding.codec: syslog`, `tls` (CA, client certificate, `verify_certificate`, `server_name`) | `syslog` (RFC 5424; `rfc3164` is reported). With the `json` or `text` codec the agent still sends RFC 5424 frames (JSON as the message for `json`) — check the receiver |
| `loki` `endpoint` (+ `path`), `tenant_id`, static `labels`, labels taken from a field of the same name (`host: "{{ host }}"`), `structured_metadata`, `encoding.codec` `text` / `json`, `compression`, `auth` basic / bearer | `loki` `endpoint`, `tenant`, `labels`, `label_fields`, `structured_metadata: true` (every remaining field, not only the listed ones), `line_format`, `gzip`, `basic_auth_user` + password placeholder or `bearer_token` placeholder |
| `splunk_hec_logs` `endpoint`, `default_token`, `index`, `sourcetype`, `source`, `compression: gzip`, `encoding.codec: json` | `hec` with a `token` placeholder, `index`, `sourcetype`, `source`, `gzip`, `event_format: json`. A templated index or sourcetype is `manual` (set the per-event `splunk.*` fields instead) |
| `elasticsearch` `endpoints`, `bulk.index` or `mode: data_stream`, `auth.strategy: basic` | `elasticsearch` `endpoints`, `index` (a date-stamped index is reported: the agent writes one index or data stream), `username` + password placeholder. The agent writes ECS-shaped documents |
| `kafka` `bootstrap_servers`, `topic`, `key_field`, `compression`, `encoding.codec` `json` / `text`, `sasl` (`PLAIN`, `SCRAM-SHA-256`, `SCRAM-SHA-512`), `tls` | `kafka` `brokers`, `topic`, `key_field`, `compression`, `encoding`, `sasl` with placeholders, `tls`. The JSON record is the agent's event shape |
| `aws_s3` `bucket`, `region`, `endpoint`, `force_path_style`, `key_prefix`, `compression`, `storage_class`, `encoding.codec` `json` / `text`, static keys | `s3` with the same settings and key placeholders. The agent writes time-partitioned NDJSON objects under the prefix; a date-templated prefix keeps its static part |
| `console`, `blackhole`, `file` | skipped |
| any sink, `-target syslog` | one `syslog` output over TLS (`-syslog-endpoint`) |

## What is not converted (and what to do)

| Vector | Why / instead |
|---|---|
| VRL beyond assignments, `del()` and renames (`parse_*`, `merge`, `if`, `abort`, functions, `file:` programs) | listed with the statement's line; port with the agent's `expr`, `parse_json`, `parse_kv`, `parse_csv`, `parse_xml` and `redact` processors |
| a remap / sample / throttle / dedupe in front of only some sinks of a source | not applied (see Topology); give that path its own source, or move the transform so it feeds every sink of the source |
| `reduce`, `aggregate`, `lua`, `log_to_metric`, `exclusive_route` and other transforms | no equivalent; the sinks behind them receive the events unchanged |
| `vector` source and sink | the agent does not speak Vector's protocol; between agents use the `relay` output and `relay_in` input, or OTLP |
| `opentelemetry`, `fluent` and `logstash` sources; `opentelemetry` and Azure sinks | the agent has `otlp_in`, `fluent_in`, `beats_in`, the `otlp` output and the `sentinel` output; configure them by hand |
| `http` and `datadog_logs` sinks | no equivalent output; use `-target syslog`, or an output the receiver also accepts |
| `http_server` with basic or custom authentication | not converted: the agent's HTTP input authenticates with a bearer token, and opening it without authentication would accept anyone |
| templated `topic`, `tenant_id`, label keys with `*`, labels taken from a differently named field | reported; split the output or rename the field with an `expr` rule |
| `elasticsearch` with AWS authentication or an ingest `pipeline`; `aws_s3` with an assumed role, a profile or a credentials file | configure by hand; the report names the setting |
| `enrichment_tables` | the agent's `lookup` processor enriches from a local table; configure by hand |
| buffers, batching, request and healthcheck settings | not needed: the agent spools to disk and retries on its own (`info`) |
| Vector's checkpoints (`data_dir`) | not needed: the agent keeps its own offsets and cursors; it does not import Vector's |

## Step 3 — cut over without a gap

Prefer a few seconds of duplicates to a gap:

1. Install the agent with the converted configuration and start it. With `start_at: end` it reads only lines
   written from now on.
2. Confirm in the destination that events arrive from the agent.
3. Stop and disable Vector (or remove the converted sources from its configuration).

Between steps 1 and 3 both send new lines, so those few seconds arrive twice. Stopping Vector *first* would lose
the lines written before the agent starts.

The events differ in shape from Vector's: the agent's event has a message, a host, a severity and a source (the
file is named in the source, `file:<path>`), and its fields travel as Loki structured metadata, Elasticsearch
`labels`, syslog structured data or Kafka / S3 JSON fields. Queries and dashboards that read Vector's field names
need a look.

## Sending to logrok instead

```bash
luna-migrate vector /etc/vector/vector.yaml -out agent.yaml \
  -target syslog -syslog-endpoint logrok.example.com:16575
```

Sources and transforms convert as above; only the destination changes: the one syslog output receives the events
any of Vector's sinks received. Add the TLS trust (`ca_file`, and a client certificate if your receiver requires
one) to the generated `syslog` output.

## Licensing

The converter is free. What the converted configuration uses follows the normal rules
([LICENSING](../LICENSING.md)): file tailing, journald, syslog and HTTP input, the parse and `expr` processors and
one RFC 5424 syslog destination over TCP or TLS are Core; the `loki`, `hec`, `elasticsearch`, `kafka` and `s3` outputs (every
destination other than RFC 5424 syslog), more than one destination and per-destination routing conditions, the
`sample`, `throttle` and `dedup` processors, the Splunk HEC receiver, container-log mode, the Windows Event Log
input, a UDP syslog destination (a `socket` sink with `mode: udp`) and the disk spool are Apex.
