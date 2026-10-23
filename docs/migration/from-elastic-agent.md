# Migrating from Elastic Agent

The agent can take over a host's log collection from a **standalone** Elastic Agent and keep delivering to the
same place: Elasticsearch or OpenSearch, Logstash (it speaks the Beats protocol), or Kafka. Or it can send the
same data as RFC 5424 syslog to logrok or any syslog receiver.

You don't rewrite the policy by hand: the agent reads `elastic-agent.yml` (and the input files in `inputs.d`
beside it) and writes an agent configuration plus a report of everything it could not carry over.

## Is it a fit?

**Verified against:** the Fleet and Elastic Agent Guide 8.19 (standalone policies). A policy was loaded by Elastic Agent 8.15.3 where its image was available.

Check these first; they decide whether the conversion is a drop-in or a project.

| If your Elastic Agent… | Then |
|---|---|
| is **standalone** and collects files (`filestream`, `logfile`, `log`), journald, syslog/TCP/UDP, containers' log files, or Windows event logs (`winlog`) | converted (tables below) |
| is **managed by Fleet** | **its policy lives in Kibana, not in `elastic-agent.yml`**, so there is nothing to convert on the host. Download the policy from Kibana (Fleet → Agents → Add agent → *Run standalone* → *Download Policy*; the wording can differ between versions), save it as a YAML file and convert that file. The converter stops with this explanation when it finds an enrolled agent's file that has no `inputs:`; if the file does list inputs, they are converted and the report carries a `manual` finding telling you to download the current policy |
| uses **integrations** (nginx, system, Windows, …) | the inputs convert, but each integration also ships Elasticsearch ingest pipelines, index templates and dashboards that parse the lines: that parsing is **not reproduced** (the report says so for every integration input). Keep the integration's assets in Elasticsearch if you want the parsed fields |
| collects **metrics** (`system/metrics`, …) or runs endpoint security, APM, synthetics, or Fleet Server | not converted: the agent forwards logs |
| sends to a destination other than Elasticsearch, Logstash or Kafka | not converted natively; use `-target syslog` or configure the destination by hand |

## How it fits together

```
before:   files / journal / sockets / event logs ──► Elastic Agent ──► Elasticsearch | Logstash | Kafka
after:    files / journal / sockets / event logs ──► agent         ──► the same destinations (bulk API, Beats protocol, Kafka)
          (or)                                      agent          ──► RFC 5424 syslog over TLS (logrok / any syslog receiver)
```

Elastic Agent names its destinations (`outputs:`) and points each input at one with `use_output`. The converter
keeps that: every input goes to the output it names.

## Step 1 — convert

Get `luna-migrate` first: a separate download published with every agent release (`luna-migrate-<os>-<arch>`, saved as `luna-migrate` / `luna-migrate.exe`) — see [Getting luna-migrate](../USER-GUIDE.md#getting-luna-migrate).

Run the converter on the Elastic Agent host (it only reads the policy):

```bash
luna-migrate elastic-agent /opt/Elastic/Agent/elastic-agent.yml \
  -out /etc/logrok-universal-agent/agent.yaml \
  -report elastic-agent-conversion.json
```

```powershell
& ".\luna-migrate.exe" elastic-agent "C:\Program Files\Elastic\Agent\elastic-agent.yml" `
  -out "C:\ProgramData\logrok-universal-agent\agent.yaml" `
  -report elastic-agent-conversion.json
```

The input files in the `inputs.d` directory next to the policy file are read as well (`*.yml` and `*.yaml`). When
you convert a copy of the policy, copy `inputs.d` beside it.

| Flag | Default | Meaning |
|---|---|---|
| `-out` | *(required)* | where to write the agent configuration; an existing file is never overwritten |
| `-report` | — | also write the full report as JSON |
| `-target` | `native` | `native`: send where Elastic Agent sent (Elasticsearch → `elasticsearch`, Logstash → `beats`, Kafka → `kafka`). `syslog`: RFC 5424 over TLS |
| `-syslog-endpoint` | — | `host:port` of the syslog receiver for `-target syslog` |
| `-syslog-plain` | off | send the syslog output over plain TCP instead of TLS (a receiver without TLS, or a lab) |
| `-start-at` | `end` | where file inputs start on first run: `end` (only new lines: nothing Elastic Agent already shipped is shipped again) or `beginning` |

luna-migrate checks the configuration's structure, then runs the installed agent's `-check-config` (found beside `luna-migrate` or on the `PATH`, or named with `-agent`), which builds every module exactly as that agent version would; a configuration the agent rejects is removed. Without an agent to hand it says so — then run `logrok-universal-agent -check-config -config agent.yaml` on the target host.

## Step 2 — read the report

| Severity | Meaning |
|---|---|
| `manual` | **not carried over**: a person must decide (each line says what to do instead) |
| `warning` | carried over, with a difference to check |
| `info` | carried over differently, nothing to do |
| `skipped` | deliberately not converted: inputs, streams and outputs that are switched off (`enabled: false`, `condition: false`) or that nothing uses |

Every line names the input and stream (`inputs[0].streams[1]`) and the file and line of the setting in the policy.

Then fill in what the report lists under "fill in before deploying". API keys, passwords, key passphrases and
credentials inside host URLs are **never copied** out of the policy: the generated file says `REPLACE_WITH_…`
(search for it). Variables such as `${VAR}` and `${provider.key}` are never expanded either — the report shows
every one it met.

## What is converted

### Inputs and streams

Each **stream** becomes one agent input (a stream has its own paths, processors and dataset). An input without
`streams:` (settings directly on the input) is one stream.

| Elastic Agent | Agent |
|---|---|
| `filestream`, `log` / `logfile` / `event/file` — `paths` | `filetail` `paths`. `**` must be a whole path segment |
| `prospector.scanner.exclude_files`, `exclude_files` | `exclude` globs when the expression is a file-extension match (`\.gz$`); any other expression is `manual` |
| multiline (`parsers: - multiline:` or `multiline.*`) | `multiline.line_start_pattern` / `line_end_pattern` (the same rules as [Filebeat](from-filebeat.md#inputs)) |
| `include_lines` / `exclude_lines` | exact: only that stream's events are dropped |
| `encoding`, `tail_files` | as for Filebeat |
| `container`, `docker`, `log/docker` | `filetail` with `format: auto`, `docker` or `cri` |
| `journald` (`units`, `syslog_identifiers`) | `journald` |
| `syslog` / `log/syslog` (`protocol.udp`, `protocol.tcp`) | `syslog_in`, one per protocol |
| `tcp` / `event/tcp`, `udp` | `syslog_in` |
| `winlog` (`name`, `event_id`) | `windows_eventlog`; the `event_id` allow/block list becomes a structured query |
| `enabled: false`, `condition: false` | skipped |
| `data_stream.dataset`, `data_stream.namespace`, `data_stream.type` | carried on every event as the fields `data_stream.dataset`, `data_stream.namespace`, `data_stream.type` (defaults `generic`, `default`, `logs`). For an Elasticsearch output they also choose the destination, see below |
| `use_output` | the stream goes to the output of that name (default: `default`) |
| `fields`, `fields_under_root`, `tags`, `processors` (`add_fields`, `drop_event`, `drop_fields`, `rename`) | as for [Filebeat](from-filebeat.md#fields-tags-and-processors): processors are applied **only to their own stream** (a helper field `beats_input` scopes them and is removed again) |

### Outputs

| Elastic Agent | Agent |
|---|---|
| `type: elasticsearch` | `elasticsearch` output(s): `hosts` → `endpoints`, `api_key` / `username` / `password` → placeholders to fill in, `ssl` → `ca_file`, `path`. **One output per data stream:** each stream is written to `<type>-<dataset>-<namespace>` (for example `logs-nginx.access-prod`), the same data stream Elastic Agent wrote to |
| `type: logstash` | `beats` output: `hosts` → `endpoint` / `endpoints` (default port 5044), `ssl` → `tls`, `ca_file`, `cert_file`, `key_file`. Logstash receives `data_stream.type`, `data_stream.dataset` and `data_stream.namespace` as fields (Logstash shows them as a nested `data_stream` object) |
| `type: kafka` | `kafka` output: `hosts` → `brokers`, `topic`, `username` / `password` + `sasl.mechanism` → `sasl`, `ssl`, `compression` |
| `ssl.verification_mode: none` | `insecure_skip_verify: true` with a `warning` |
| any output, `-target syslog` | one `syslog` output over TLS (`-syslog-endpoint`); the `data_stream.*` fields travel as structured data |

When the result has more than one destination (several outputs, or one Elasticsearch output with several data
streams) each input carries a helper field `ea_output` and each output a `when:` condition on it. The routing runs
after the processors, so the helper field is delivered with every event: strip it in the receiver if you don't
want it. The report has a warning for this. A policy with one output and one data stream needs none of it.

## What is not converted (and what to do)

| Elastic Agent | Why / instead |
|---|---|
| a **Fleet-managed** agent's file | no policy in it: download the policy from Kibana and convert that file (see "Is it a fit?") |
| `condition:` expressions on inputs, streams or processors (`${host.platform} == 'linux'`) | they are evaluated by Elastic Agent on its host at run time. The input is converted as if the condition were true and the report says so: delete the input if it does not apply to this host. `true` / `false` are decided |
| `${…}` variables and `providers` (`env`, `local`, `local_dynamic`, `host`, …) | not expanded. In `paths` and `hosts` the entry is dropped and reported; elsewhere the text is copied literally and reported: set the real value |
| integrations' ingest pipelines (`pipeline: logs-…`), index templates, dashboards | the agent does not select a pipeline or install templates |
| `system/metrics` and other metrics, endpoint, APM, synthetics, Fleet Server inputs | not converted: the agent forwards logs |
| `unix` input, `ndjson` parser / `json.*`, `dissect`, `script`, `add_*_metadata` and other processors without an exact equivalent | `manual` lines say what to do instead, as for [Filebeat](from-filebeat.md#what-is-not-converted-and-what-to-do) |
| `ca_sha256` (pinned CA fingerprint), `proxy_url`, output `headers` / `parameters` | not carried over: set `ca_file` so the server certificate is verified against your CA; configure a proxy on the host |
| `agent.monitoring`, `agent.logging`, `agent.download`, `fleet`, … | Elastic Agent's own runtime settings. The agent has its own logging, metrics and upgrade; its self-monitoring data is not collected |
| `remote_elasticsearch` and other output types | `manual`; use `-target syslog` or define the destination by hand |
| the Elastic Agent state (file offsets) | the agent keeps its own file state (it does not import Elastic Agent's offsets) |

The Elasticsearch document differs from Elastic Agent's: the agent writes ECS-shaped documents with its own
`agent.*`, `event.*` and `log.*` fields and puts your fields (including the `data_stream.*` values) under `labels`
with dots replaced by `_` (`labels.data_stream_dataset`). Dashboards built on Elastic Agent's exact field set need
a look.

## Step 3 — cut over without a gap

Prefer a few seconds of duplicates to a gap:

1. Install the agent with the converted configuration and start it. With `start_at: end` it reads only lines
   written from now on.
2. Confirm in the destination that events arrive from the agent: the same data streams (or topics), with your fields.
3. Stop and disable Elastic Agent.

Between steps 1 and 3 both send new lines, so those few seconds arrive twice. Stopping Elastic Agent *first* would
lose the lines written before the agent starts.

## Sending to logrok instead

```bash
luna-migrate elastic-agent /opt/Elastic/Agent/elastic-agent.yml -out agent.yaml \
  -target syslog -syslog-endpoint logrok.example.com:16575
```

Inputs, fields and processors convert as above; only the destination changes. Add the TLS trust (`ca_file`, and a
client certificate if your receiver requires one) to the generated `syslog` output.

## Licensing

The converter is free. What the converted configuration uses follows the normal rules
([LICENSING](../LICENSING.md)): file tailing, journald, syslog input and syslog forwarding are Core; the
`elasticsearch`, `beats` and `kafka` outputs, container log parsing, the Windows Event Log input and the disk
spool are Apex.
