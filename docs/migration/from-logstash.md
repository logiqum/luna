# Migrating from Logstash

Logstash is usually not a host agent: it is a **central pipeline** that Beats, syslog senders and applications
feed, and that parses events (grok, date, ruby…) before indexing them. So decide first which of two moves you
are making:

- **Logstash runs on the hosts and mostly ships files or syslog** (a forwarder role). Convert its pipeline: the
  agent reads the same files, listens on the same ports and delivers to the same destinations. This page is
  mostly about that.
- **Logstash is your central tier**, fed by Beats. Usually you keep it, and the agent takes the edge: its
  `beats_in` input ([configuration reference](../CONFIGURATION.md))
  receives what Beats send to Logstash today (point `output.logstash.hosts` at the agent), and its `beats` output
  sends to a Logstash `beats` input. The agent then becomes the **gateway** — buffering to disk, TLS, reduction —
  in front of Logstash or instead of it, while the parsing you cannot carry stays where it is.

```
forwarder role:  files / syslog ──► Logstash ──► elasticsearch | kafka | syslog | s3
          after: files / syslog ──► agent    ──► the same destinations, selected by a condition per output

central role:    Beats ──► Logstash (grok, date, …) ──► Elasticsearch
          after: Beats ──► agent beats_in (spool, TLS, reduction) ──► Logstash beats input ──► Elasticsearch
            (or) Beats ──► agent beats_in ──► elasticsearch / kafka / syslog directly, when no parsing is needed
```

## Is it a fit?

**Verified against:** the Logstash Reference 8.19. A pipeline was accepted by Logstash 8.15.3 and delivery into Logstash 8.15.3 was run end to end.

| If your pipeline… | Then |
|---|---|
| reads files (`file`), or receives Beats / Elastic Agent, syslog, TCP, UDP or HTTP | converted (tables below) |
| uses `if` / `else if` / `else` conditionals, `drop`, `mutate` (add / rename / replace / remove fields), `json`, `kv`, `csv` | converted, with the limits below |
| relies on **grok**, **dissect**, **date**, **ruby**, **translate**, **geoip** or other enrichment filters | **not converted**: the agent has no equivalent. Keep that parsing downstream (keep Logstash as the central tier) |
| sends to Elasticsearch, Kafka, syslog, TCP / UDP or S3 | converted |
| sends with the `http`, `file` or `pipeline` outputs, or reads Kafka | not converted; see below |

## Step 1 — convert

Get `luna-migrate` first: a separate download published with every agent release (`luna-migrate-<os>-<arch>`, saved as `luna-migrate` / `luna-migrate.exe`) — see [Getting luna-migrate](../USER-GUIDE.md#getting-luna-migrate).

Run the converter where the Logstash configuration is (it only reads files):

```bash
# a settings directory with pipelines.yml, a pipelines.yml, a conf.d directory or one .conf file
luna-migrate logstash /etc/logstash \
  -out /etc/logrok-universal-agent/agent.yaml \
  -report logstash-conversion.json
```

```powershell
& ".\luna-migrate.exe" logstash "C:\logstash\config\pipelines.yml" `
  -out "C:\ProgramData\logrok-universal-agent\agent.yaml" `
  -report logstash-conversion.json
```

| Flag | Default | Meaning |
|---|---|---|
| `-out` | *(required)* | where to write the agent configuration; an existing file is never overwritten |
| `-report` | — | also write the full report as JSON |
| `-target` | `native` | `native`: send where Logstash sent. `syslog`: everything to one RFC 5424 receiver over TLS |
| `-syslog-endpoint` | — | `host:port` of the syslog receiver for `-target syslog` |
| `-syslog-plain` | off | send the syslog output over plain TCP instead of TLS (a receiver without TLS, or a lab) |
| `-start-at` | `end` | where file inputs start on first run: `end` (only new lines: nothing Logstash already shipped is shipped again) or `beginning` |

**What is read.** A directory is read the way `logstash -f <directory>` reads it: **every file** in it, concatenated
in name order, as one pipeline. A `pipelines.yml` (or a settings directory that holds one) is read pipeline by
pipeline: `path.config` (a file, a directory or a glob, with `*`, `**` and `{a,b}`) or `config.string`. A relative
`path.config` is read beside `pipelines.yml`. A pipeline that cannot be read is reported and the others still
convert.

**Environment variables and the keystore.** The agent does not expand `${VAR}` or `${VAR:default}`, and neither
does `luna-migrate`. An address read from a variable becomes `REPLACE_WITH_…`; a setting that needs the value
(a port, a path, a field value) is not converted and the report names the variable.

luna-migrate checks the configuration's structure, then runs the installed agent's `-check-config` (found beside
`luna-migrate` or on the `PATH`, or named with `-agent`), which builds every module exactly as that agent version
would; a configuration the agent rejects is removed. Without an agent to hand it says so — then run
`logrok-universal-agent -check-config -config agent.yaml` on the target host.

## Step 2 — read the report

| Severity | Meaning |
|---|---|
| `manual` | **not carried over**: a person must decide (each line says what to do instead) |
| `warning` | carried over, with a difference to check |
| `info` | carried over differently, nothing to do |
| `skipped` | deliberately not converted: `stdout` / `null` outputs, test inputs (`stdin`, `generator`) |

Each finding names the file and line it came from (`/etc/logstash/conf.d/20-filter.conf:14`) and the plugin
(`filter mutate (my_id)`; with several pipelines, `pipeline apps: …`). Then fill in what the report lists under
"fill in before deploying". Passwords, API keys, SASL passwords and AWS keys are **never copied** (a SASL user name is): the generated
file says `REPLACE_WITH_…` (search for it). Every setting the converter did not read is reported, never dropped
silently.

## Conditionals, filters and routing

A Logstash pipeline sends every event of its inputs through its filters, in order, then to its outputs; the
agent has the same shape: inputs, one processing chain, outputs.

- A **conditional filter** becomes an `expr` rule with that condition (`if:`); `else if` and `else` become the
  negation of the earlier branches joined with their own condition, so each branch sees exactly the events it saw
  in Logstash.
- A **conditional output** becomes an output with a `when:` condition. Logstash evaluates output conditionals after
  the filters; so does the agent. Per-output conditions and more than one output are Apex features.
- **Several pipelines** (`pipelines.yml`) are kept apart: every input stamps the field `logstash.pipeline` (its
  pipeline id), and every rule and output of a pipeline is conditioned on it — a drop in one pipeline never touches
  another pipeline's events. The field is delivered with the events. A pipeline whose outputs could not be
  converted loses its inputs too (the report says so). With a single pipeline there is no such field.

Conditions are converted only when they mean the same on the agent:

| Logstash | Agent |
|---|---|
| `[field]`, `[a][b]` | `fields.field`, `fields.a.b` — `[message]` is the event's message, `[host]` / `[host][name]` its host |
| `==` `!=` `<` `>` `<=` `>=` | the same operators |
| `=~ /re/`, `!~ /re/` | `matches` (regular expressions must be valid RE2: no lookarounds or backreferences) |
| `[f] in ["a", "b"]`, `not in` | `fields.f == "a" \|\| fields.f == "b"`, negated |
| `"x" in [f]` | `fields.f contains "x"` (on `[tags]`: a substring of the comma-separated tags — reported) |
| `and`, `or`, `xor`, `nand`, `!`, parentheses | `&&`, `\|\|`, the equivalent combinations, `!`, parentheses |
| a bare `[field]` | the field is present and not empty |

Not converted (the filter or output under it is reported and **not applied** — a drop under it drops nothing):
`[@metadata]` and `[@timestamp]`, field names with characters other than letters, digits and `_`, `%{…}`
references inside a condition, conditions that mix `and` / `or` / `xor` / `nand` **without parentheses** (Logstash
documents no precedence between them, so the converter does not guess — add parentheses), comparisons of two
constants. A condition that reads a field no converted input or filter creates (for example one grok extracted)
is converted but reported, because the agent's events will not carry that field.

## What is converted

### Inputs

| Logstash | Agent |
|---|---|
| `file` `path`, `exclude`, `stat_interval` | `filetail` `paths`, `exclude` (Logstash matched the file name; the pattern is written into each path's directory), `poll_interval`. `start_at` follows `-start-at`; the sincedb is not imported (the agent keeps its own positions). `mode => read` is converted as tailing (warning: the agent never deletes files) |
| `file` `codec => multiline` with `negate => true` and `what => previous` (or `next`) | `multiline.line_start_pattern` (or `line_end_pattern`), `auto_flush_interval` → `flush_timeout`. Other combinations and grok patterns inside the pattern are manual |
| `file` `codec => plain { charset => "UTF-16LE" }` | `encoding: utf-16le` (or `utf-16be`) |
| `beats`, `elastic_agent` | `beats_in`: `listen` (`host:port`), TLS certificate and key, client CA when client certificates were required or optional (the agent then requires them, reported as a warning when they were optional). Beats documents keep their dotted field names (`log.file.path`, `fields.app`), so conditions over them carry over |
| `syslog` | `syslog_in` on TCP and UDP (port 514 by default, as in Logstash). The agent parses RFC 3164 and RFC 5424 itself; Logstash's own fields (`priority`, `program`, …) are not created |
| `tcp`, `udp` | `syslog_in`; `tcp` reads one event per line (`framing: newline`), `udp` one event per datagram; TLS for `tcp` with the client CA required by default, as in Logstash |
| `http` | `http_in`. Basic authentication becomes a bearer token to fill in (warning: senders change their credentials). The agent makes one event per line of the body |
| common options `type`, `tags`, `add_field` (fixed values) | `add_fields` on the input: `type`, `tags` (comma-separated), and each field |
| codec `json` / `json_lines` | a `parse_json` processor — **only when every converted input uses it** (the agent's parse processors apply to every event) |

### Filters

| Logstash | Agent |
|---|---|
| `drop` | an `expr` drop rule with the branch's condition; a single `if [field] == "value"` (or `in [...]`) becomes a `filter` processor (`exclude_fields`). `percentage` is manual |
| `mutate` `rename`, `update`, `replace`, and the common `add_field`, `remove_field`, `add_tag` | `expr` rules (`rename`, `set` — `update` only where the field has a value —, `remove`), in Logstash's documented order. Values must be fixed text: `%{…}` references are manual. `add_tag` sets the `tags` field (warning: the agent cannot append to a list) |
| `json` (`source`, `target`) | `parse_json` (`source`, `prefix: <target>.`) |
| `kv` (`source`, `target`, `prefix`, single-character `field_split` / `value_split`) | `parse_kv` (`source`, `prefix`, `pair_separator`, `kv_separator`) |
| `csv` (`columns`, single-character `separator`, `source`, `target`) | `parse_csv` |

`json`, `kv` and `csv` are converted only when unconditional and in a single pipeline: the agent's parse
processors have no condition. Logstash applies a filter's `add_field` / `add_tag` only when the filter succeeds;
the agent applies them to every event the filter sees (reported).

### Outputs

| Logstash | Agent |
|---|---|
| `elasticsearch` `hosts` (plain HTTP unless TLS is on, port 9200 when none is given, as in Logstash), `index` (fixed), data stream settings, `user` + `password` placeholder or `api_key` placeholder, CA file, `ssl_verification_mode => none` | `elasticsearch`: `endpoints`, `index`, `username` / `password` / `api_key`, `ca_file`, `insecure_skip_verify`. An index computed per event (`%{+YYYY.MM.dd}`) or no index at all is reported: the agent writes one fixed index (default `logs-luna-default`) |
| `kafka` `bootstrap_servers`, `topic_id` (fixed), `codec => json`, `compression_type`, `message_key` naming one field, `security_protocol` SSL / SASL, SASL `PLAIN` / `SCRAM-SHA-256` / `SCRAM-SHA-512` (the user from `sasl_jaas_config`, the password a placeholder) | `kafka`: `brokers`, `topic`, `encoding: json` (else `message`), `compression`, `key_field`, `tls`, `sasl` |
| `syslog` `host`, `port`, `protocol` (`udp` — Logstash's default —, `tcp`, `ssl-tcp`), CA and client certificate | `syslog`: `endpoint`, `protocol`, `tls`, `ca_file`, `cert_file` / `key_file`. The agent always sends RFC 5424 (Logstash's default was RFC 3164 — warning) and verifies the receiver's certificate (Logstash did not by default — warning) |
| `tcp`, `udp` | `syslog` over TCP or UDP with `encoding: json` (`encoding: text` for a `line` / `plain` codec): the receiver gets RFC 5424 frames, not raw JSON lines (warning) |
| `s3` `bucket`, `region`, `prefix`, `endpoint`, `encoding => gzip`, `codec`, `storage_class`, `force_path_style => true` inside `additional_settings`, access keys (placeholders) | `s3`: `bucket`, `region`, `key_prefix`, `endpoint`, `compression`, `encoding`, `storage_class`, `force_path_style`, `access_key_id` / `secret_access_key` |
| `stdout`, `null` | skipped |
| any output, `-target syslog` | one `syslog` output over TLS (`-syslog-endpoint`), with a `when:` that selects what Logstash's outputs selected |

## What is not converted (and what to do)

| Logstash | Why / instead |
|---|---|
| `grok`, `dissect` | no pattern parser in the agent; `parse_kv`, `parse_csv`, `parse_json` cover structured text — otherwise keep the parsing in a central tier behind the agent |
| `date` | the agent keeps the time it read the line (files) or the sender's timestamp (syslog, Beats) |
| `ruby` | no scripting; port the logic to `expr` rules (set / remove / rename / drop by condition) |
| `translate` | the agent's `lookup` processor (Apex) enriches from a local table — port the dictionary by hand |
| `geoip`, `useragent`, other enrichment filters | no equivalent; keep them downstream |
| `mutate` `gsub`, `convert`, `lowercase`, `split`, `merge`, `copy`, `coerce`, `remove_tag` | no equivalent (to mask values, the `redact` processor, which is Apex) |
| `drop { percentage => … }` | the `sample` processor (Apex) keeps 1 in N events |
| `kafka` input | the agent has no Kafka input; keep a consumer, or have the producers send to the agent |
| `pipeline` input / output (pipeline-to-pipeline) | the agent has one chain per configuration; merge the pipelines by hand |
| `http` output | use the agent's `hec`, `sentinel`, `xsiam` or `otlp` output by hand, or `-target syslog` |
| `file` output | the agent does not write events to local files |
| `elasticsearch` `cloud_id`, `cloud_auth`, Java keystores / truststores, `document_id`, ingest `pipeline` | configure by hand; the report names the setting (an ingest pipeline can be the index's default pipeline) |
| `kafka` SASL `GSSAPI` (Kerberos, Logstash's default mechanism), `OAUTHBEARER`, AWS MSK IAM; a topic computed per event | not converted |
| `[@metadata]` fields | they exist only inside Logstash |
| persistent queue, dead letter queue, `pipeline.workers`, batch settings | not needed: the agent batches by itself and keeps a disk spool per destination |

## Step 3 — cut over without a gap

Prefer a few seconds of duplicates to a gap:

1. Install the agent with the converted configuration and start it. With `start_at: end` it reads only lines
   written from now on. A listener (Beats, syslog, TCP) needs its port: run the agent on another port first and
   move the senders, or stop Logstash's listener and start the agent right after (Beats resend what was not
   acknowledged and most TCP senders reconnect; UDP datagrams sent in that moment are lost).
2. Confirm in the destination that events arrive from the agent.
3. Stop and disable Logstash (or the pipelines you replaced).

The agent does not read the sincedb. The events differ in shape from Logstash's: the agent's event has a message, a
host, a severity and a source (the file is named in the source, `file:<path>`, not in `[log][file][path]`), and
its fields travel as Elasticsearch `labels`, syslog structured data or Kafka JSON fields. Dashboards and consumers
that read Logstash's field names need a look.

## Sending to logrok instead

```bash
luna-migrate logstash /etc/logstash -out agent.yaml \
  -target syslog -syslog-endpoint logrok.example.com:16575
```

Inputs, conditionals and filters convert as above; only the destination changes. Add the TLS trust (`ca_file`, and a
client certificate if your receiver requires one) to the generated `syslog` output.

## Licensing

The converter is free. What the converted configuration uses follows the normal rules
([LICENSING](../LICENSING.md)): file tailing, the syslog and HTTP inputs, the parse, `filter` and `expr` processors
and one RFC 5424 syslog destination are Core; the Beats input, the `elasticsearch`, `kafka` and `s3` outputs, UDP
syslog output, more than one destination, per-destination routing conditions and the disk spool are Apex.
