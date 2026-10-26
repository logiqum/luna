# Migrating from Fluentd (td-agent, fluent-package)

The agent can take over a host's log collection from Fluentd — whether installed as **td-agent** or as its
successor **fluent-package** — and keep delivering to the same place: a Fluentd or Fluent Bit aggregator (Forward
protocol), Elasticsearch or OpenSearch, Kafka, Loki, Splunk (HTTP Event Collector), an S3 bucket or a syslog
receiver. Or it can send the same data as RFC 5424 syslog to logrok or any syslog receiver.

Why now: the Fluentd project gave **td-agent 4** a scheduled end of life of December 2023, and stopped
maintaining **fluent-package 5** on 31 December 2025 (v5.0.9 is the last release of that series; the project
recommends fluent-package 6). A host still on either can move to the agent instead of to the next Fluentd package.

You don't rewrite the configuration by hand: the agent reads the Fluentd configuration
(`/etc/fluent/fluentd.conf` for fluent-package, `/etc/td-agent/td-agent.conf` for td-agent), follows its
`@include` lines, and writes an agent configuration plus a report of everything it could not carry over.

If you would rather leave Fluentd in place, the agent can also sit behind it: point Fluentd's `forward` output at
the agent's `fluent_in` input and the agent adds buffering, TLS and its other destinations.

## Is it a fit?

**Verified against:** Fluentd 1.x documentation (read 2026-10-16; its pages reference releases up to v1.19). A configuration was accepted by Fluentd 1.18 and delivery was checked against Fluentd 1.18.

Check these first; they decide whether the conversion is a drop-in or a project.

| If your Fluentd… | Then |
|---|---|
| tails files (`in_tail`), listens for syslog / Forward / HTTP, reads the systemd journal or the Windows event log | converted (tables below) |
| routes by tag with `<match>`, `<filter>`, `<label>`, `relabel` and `copy` | converted: the agent routes with a condition per output (see [Tags and routing](#tags-and-routing)) |
| uses `grep`, static `record_transformer` fields, or the `parser` filter with JSON | converted |
| parses with the **regexp**, apache / nginx, csv / ltsv or syslog parsers | **not converted** — each line arrives as the message (JSON and `none` are converted; `multiline` keeps its record boundaries) |
| uses **Ruby** — `enable_ruby`, `${record[...]}` expressions, `"#{...}"` in values | **not converted.** A `record_transformer` value with `"#{...}"` leaves that field out and is reported `manual`, so you add it by hand; nothing is ever evaluated |
| uses `rewrite_tag_filter`, `record_modifier`, `kubernetes_metadata`, `concat`, `geoip` or other filters / outputs not listed below | not converted; the report names each one |
| writes to `file`, the generic `http` output, CloudWatch, BigQuery and similar | not converted natively; use `-target syslog` or configure the destination by hand |

## How it fits together

```
before:   files / journal / sockets ──► Fluentd ──► forward | elasticsearch | kafka2 | loki | splunk_hec | s3 | …   (first <match>)
after:    files / journal / sockets ──► agent   ──► the same destinations, selected by a condition per output
          (or)                          agent   ──► RFC 5424 syslog over TLS (logrok / any syslog receiver)
```

## Step 1 — convert

Get `luna-migrate` first: a separate download published with every agent release (`luna-migrate-<os>-<arch>`, saved as `luna-migrate` / `luna-migrate.exe`) — see [Getting luna-migrate](../USER-GUIDE.md#getting-luna-migrate).

Run the converter on the Fluentd host (it only reads the configuration files):

```bash
luna-migrate fluentd /etc/fluent/fluentd.conf \
  -out /etc/logrok-universal-agent/agent.yaml \
  -report fluentd-conversion.json
```

```powershell
& ".\luna-migrate.exe" fluentd "<path to your fluentd.conf>" `
  -out "C:\ProgramData\logrok-universal-agent\agent.yaml" `
  -report fluentd-conversion.json
```

| Flag | Default | Meaning |
|---|---|---|
| `-out` | *(required)* | where to write the agent configuration; an existing file is never overwritten |
| `-report` | — | also write the full report as JSON |
| `-target` | `native` | `native`: send where Fluentd sent. `syslog`: everything to one RFC 5424 receiver over TLS |
| `-syslog-endpoint` | — | `host:port` of the syslog receiver for `-target syslog` |
| `-syslog-plain` | off | send the syslog output over plain TCP instead of TLS (a receiver without TLS, or a lab) |
| `-start-at` | `end` | where file inputs start on first run: `end` (only new lines: nothing Fluentd already shipped is shipped again) or `beginning`. An `in_tail` with `read_from_head true` starts at the beginning |

`@include` lines are followed: a relative path is relative to the including file, and a wildcard pattern reads its
files in alphabetical order, as Fluentd does. Every file read is listed in the report; an included URL, a file
that is not there, or a file that includes itself is reported instead. luna-migrate checks the configuration's
structure, then runs the installed agent's `-check-config` (found beside `luna-migrate` or on the `PATH`, or named
with `-agent`), which builds every module exactly as that agent version would; a configuration the agent rejects
is removed. Without an agent to hand it says so — then run `logrok-universal-agent -check-config -config
agent.yaml` on the target host.

## Step 2 — read the report

| Severity | Meaning |
|---|---|
| `manual` | **not carried over**: a person must decide (each line says what to do instead) |
| `warning` | carried over, with a difference to check |
| `info` | carried over differently, nothing to do |
| `skipped` | deliberately not converted: `stdout` and `null` outputs, the `@ERROR` and `@FLUENT_LOG` labels |

Each finding names the file and line it came from. Then fill in what the report lists under "fill in before
deploying". Shared keys, passwords, tokens, API keys and AWS keys are **never copied** out of the Fluentd
configuration: the generated file says `REPLACE_WITH_…` (search for it). A value written as embedded Ruby
(`"#{ENV['LOG_DIR']}"`, `"#{Socket.gethostname}"`) is never evaluated: it becomes a `REPLACE_WITH_…` placeholder
too, so put in the value it evaluates to on that host. Key passphrases are not carried over; use an unencrypted
key file the agent can read.

## Tags and routing

Fluentd stamps every event with a **tag** at its `<source>`. In its routing context (the top level, or the
`<label>` the source names with `@label`) the event passes, top to bottom, through every `<filter>` whose pattern
matches its tag, and stops at the **first** `<match>` whose pattern matches: a later `<match>` never sees it, and a
`<filter>` placed after that `<match>` never applies to it. An event no `<match>` captures is discarded. The agent
has no tags. The converter keeps the behaviour like this:

- It walks every input's events through the configuration in Fluentd's order and works out which `<match>`
  captures them — following `relabel` into labels and fanning out over every `<store>` of a `copy`.
- Patterns follow Fluentd's rules: `*` is one tag part (`app.*` matches `app.web`, not `app` or `app.web.x`),
  `**` is zero or more parts (`app.**` matches `app` too), `{a,b}` is either, several patterns separated by spaces
  match any of them, and `/…/` is a regular expression (RE2 syntax; a Ruby-only construct is `manual`).
- A destination that receives only some inputs gets a `when:` condition; one that receives all of them gets
  none. The condition tests the field `fluent.tag`, except for an `in_http` input (it tests the input's name)
  and a Forward input without a fixed tag (it tests that the event came from a Forward connection). Routing conditions are an Apex feature, as is more than one output.
- Where a condition, the `fluent` output or `include_tag_key` needs it, the converted input carries its tag in that
  field (`add_fields: {fluent.tag: web.nginx}`). The `fluent` output
  sends it as the Forward tag, so a downstream Fluentd sees the same tag; it is not part of the record. Other
  outputs carry it as an ordinary field.
- A **filter** is applied only to the inputs whose events reach it: its rules are conditioned on the same field.
- A **Forward** input keeps the sender's tag, so its conditions are evaluated per event at run time — including
  "not captured by an earlier `<match>`".
- An input whose events **no converted destination receives** is not converted; the report says which.

Where the tag cannot be known in advance the report says so and chooses the safe side (the input is delivered to
the destination rather than dropped):

| Situation | What happens |
|---|---|
| `tag app.*` on `in_tail` (Fluentd puts the file path in the tag: `/var/log/app.log` → `app.var.log.app.log`) | with a single `path` and no wildcard in it the expanded tag is used exactly; with a wildcard, or several comma-separated paths, every `<match>` is checked against all the tags the path can produce. When a pattern matches only some of them, the whole input goes to that destination (reported), and the agent stamps the fixed start of the tag |
| `in_syslog` (tags are `<tag>.<facility>.<priority>`) | the same check; the agent stamps `<tag>` and keeps facility and severity on the event (usable in conditions) |
| `in_http` (the tag is the URL path) | delivered to the first destination whose pattern could match; check it receives what you intended |
| `add_tag_prefix` on `in_forward` | not applied: conditions see the sender's tag without the prefix (reported) |

## What is converted

### Inputs

| Fluentd | Agent |
|---|---|
| `in_tail` `path` (comma-separated, wildcards), `exclude_path` | `filetail` `path` / `paths`, `exclude`. `pos_file` is not read: the agent keeps its own resume state (`state_path`). A strftime `%Y`-style path becomes a wildcard (reported) |
| `read_from_head true` | `start_at: beginning`; otherwise the `-start-at` setting |
| `<parse> @type none` | the line is the message |
| `<parse> @type json` | a `parse_json` processor (the JSON keys become fields; a line that is not JSON is kept as text) |
| `<parse> @type multiline` `format_firstline` | `multiline.line_start_pattern` (`multiline_flush_interval` → `flush_timeout`); the record is the message — the `formatN` field extraction is not converted |
| `encoding` / `from_encoding` UTF-16LE / UTF-16BE | `encoding` |
| `in_syslog` (`port`, `bind`, `<transport udp / tcp / tls>`, `frame_type octet_count`) | `syslog_in` (`listen` / `tls_listen`, `protocols`, `framing`, TLS certificate, key and client CA); RFC 5424 and 3164 are both parsed |
| `in_forward` (`port`, `bind`, `<security> shared_key`, `<transport tls>`) | `fluent_in` (`listen`, `shared_key` placeholder, TLS certificate and key, client CA); `tag` overrides the tag |
| `in_http` | `http_in`. Fluentd takes JSON / msgpack records and the tag from the URL path; the agent's input takes newline-delimited lines at one path, so JSON bodies arrive as text (add `parse_json`) |
| `systemd` (the systemd input plugin) with `matches` on `_SYSTEMD_UNIT` / `SYSLOG_IDENTIFIER` | `journald` `units` / `identifiers`. The agent keeps its own journal cursor |
| `windows_eventlog2` (and the older `windows_eventlog`): `channels`, `<subscribe>` channels, `event_query`, `render_as_xml` | `windows_eventlog` (`channels`, `query`, `render: xml`) |

### Filters

| Fluentd | Agent |
|---|---|
| `grep` `<regexp>` / `<exclude>`, `<and>` / `<or>`, the older `regexpN` / `excludeN` | `expr` drop rules scoped to the inputs the filter receives. A missing key fails a `<regexp>` and passes an `<exclude>`, as in Fluentd. Keys are the message (`message` for `<parse> @type none`), a JSON field, or a record accessor such as `$.kubernetes.labels.app` |
| `record_transformer` `<record>` with static values, `${tag}` and `${tag_parts[N]}`; `remove_keys` | `expr` set / remove rules per input (the tag placeholders resolve per input). Other placeholders and `enable_ruby` are `manual` |
| `parser` with `<parse> @type json` | `parse_json`. This processor applies to every input; when the filter receives only some inputs it is `manual` rather than applied too widely |

### Outputs

| Fluentd | Agent |
|---|---|
| `forward` | `fluent` output: `<server>` `host` / `port` → `endpoint` (further servers and `standby` servers → failover `endpoints`), `<security> shared_key` → placeholder, `require_ack_response` → `require_ack`, `compress gzip`, `transport tls` with `tls_cert_path` / client certificate |
| `elasticsearch`, `elasticsearch_data_stream` | `elasticsearch`: `host` / `port` / `scheme` or `hosts` → `endpoints`, `index_name` / `data_stream_name`, `user` / `password` / `api_key` → placeholders, `ca_file`, `ssl_verify false` → `insecure_skip_verify`. `logstash_format` writes one index (the prefix) instead of one per day |
| `kafka2` | `kafka`: `brokers`, `default_topic`, SASL (`username` / `password` placeholders, `scram_mechanism`), TLS (`ssl_ca_cert`, client certificate), `compression_codec`. The Kafka value is the agent's JSON event, not Fluentd's record JSON |
| `loki` | `loki`: `url`, `tenant`, `extra_labels`, `<label>` keys → `label_fields`, `line_format json`, basic auth as placeholders, `ca_cert`, `insecure_tls` |
| `splunk_hec` | `hec`: `protocol` / `hec_host` / `hec_port`, `hec_token` placeholder, `index`, `source`, `sourcetype`, `gzip_compression`, `ca_file`, `insecure_ssl` |
| `s3` | `s3`: `s3_bucket`, `s3_region`, `s3_endpoint`, `store_as gzip` (the plugin's default), `storage_class`, the literal start of `path` → `key_prefix`. Credentials are placeholders (the agent needs them in its configuration or environment) |
| `remote_syslog` | `syslog`: `host` / `port`, `protocol udp` / `tcp`, `tls`, `ca_file`. RFC 5424 is always sent |
| `copy` | every `<store>` becomes its own destination receiving the same events |
| `relabel` | routing continues in the named label |
| `stdout`, `null` | skipped |
| any output, `-target syslog` | one `syslog` output over TLS (`-syslog-endpoint`), with a `when:` limiting it to the events Fluentd's matches captured, only when those matches do not cover every event |

## What is not converted (and what to do)

| Fluentd | Why / instead |
|---|---|
| regexp, apache, nginx, csv, ltsv, syslog parsers on `in_tail`; `formatN` of the multiline parser | no regular-expression parse processor; each line (or multiline record) is the message — structure it at the receiver, or with `parse_kv` / `parse_csv` where the shape allows |
| embedded Ruby (`"#{...}"`), `enable_ruby`, `${record[...]}` and `${hostname}` placeholders | never evaluated; fill in the value, or set the field at the receiver |
| `rewrite_tag_filter`, `route`, and other plugins that re-tag events | they re-route by record content; express the routing with output `when:` conditions by hand |
| `record_modifier`, `kubernetes_metadata`, `concat`, `geoip`, `throttle` and other filters | not converted; the report names each one (for `throttle`, the agent has its own `throttle` processor to configure by hand) |
| `file`, `http`, `secondary_file` outputs; `<secondary>` | the agent forwards (no local file output) and spools + retries instead of a secondary output |
| `<format>` and `<inject>` on outputs | the destination receives the agent's own record shape |
| `ignore_if_prev_success` stores (fallbacks) | the agent sends to every destination |
| `path_key`, `limit_recently_modified` | the path is the event's source (`file:<path>`); no file-age filter |
| systemd matches on other fields, or a unit OR an identifier | the agent filters by units and identifiers (combined with AND); use two inputs |
| `in_forward` user authentication, per-client keys, generated certificates | shared key and mTLS are supported |
| `elasticsearch` `cloud_id`, `pipeline`, `id_key`, client certificates; per-record topics and partition keys on `kafka2`; `<label>` keys renamed on `loki` | configure by hand; the report says what each needs |
| the `@ERROR` and `@FLUENT_LOG` labels | the agent spools and retries a failed delivery; its own log is not an event stream |
| `<system>` settings, `<buffer>`, workers, flush and retry tuning | not needed: the agent has its own spool and delivery settings (reported as `info`) |
| Fluentd's `pos_file`, the journal cursor and event-log positions | the agent keeps its own state; it does not import Fluentd's positions |

## Step 3 — cut over without a gap

Prefer a few seconds of duplicates to a gap:

1. Install the agent with the converted configuration and start it. With `start_at: end` it reads only lines
   written from now on.
2. Confirm in the destination that events arrive from the agent, with your tags (or the fields that replace
   them).
3. Stop and disable Fluentd (`fluentd` for fluent-package, `td-agent` for td-agent).

Between steps 1 and 3 both send new lines, so those few seconds arrive twice. Stopping Fluentd *first* would lose
the lines written before the agent starts. The agent does not read Fluentd's position files.

The events differ in shape from Fluentd's: the agent's record has a message, a host, a severity and a source
(named `message`, `host`, `level` and `source` in the `fluent` output, `severity` in the Kafka, S3 and HEC
outputs; Loki carries it only when its `line_format` is `json`, since the default line is just the message), plus the event's fields (and, to Elasticsearch, an ECS-shaped document with your fields under
`labels`). Consumers that expect Fluentd's exact record keys need a look.

## Sending to logrok instead

```bash
luna-migrate fluentd /etc/fluent/fluentd.conf -out agent.yaml \
  -target syslog -syslog-endpoint logrok.example.com:16575
```

Inputs and filters convert as above; only the destination changes. Add the TLS trust (`ca_file`, and a client
certificate if your receiver requires one) to the generated `syslog` output.

## Licensing

The converter is free. What the converted configuration uses follows the normal rules
([LICENSING](../LICENSING.md)): file tailing, journald, syslog input, HTTP input, the parse and `expr` processors
and one RFC 5424 syslog destination over TCP/TLS are Core; UDP syslog (the `remote_syslog` default) is Apex; the `fluent`, `elasticsearch`, `kafka`, `loki`, `hec` and `s3`
outputs, more than one destination and per-destination routing conditions, the Forward receiver, the Windows
Event Log input and the disk spool are Apex.
