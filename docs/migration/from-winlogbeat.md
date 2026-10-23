# Migrating from Winlogbeat to logrok-universal-agent

## Why you're probably reading this

- **Elastic plans to retire Winlogbeat in favor of Filebeat's `winlog` input.** Its tracking issue
  ([elastic/beats#40074](https://github.com/elastic/beats/issues/40074), opened July 2024) proposes
  deprecating Winlogbeat because the Filebeat input now covers the same ground. Winlogbeat still ships
  today and its documentation carries no deprecation notice yet — check Elastic's current release notes
  for the date that applies to you.
- **Elastic Agent requires a Fleet Server** (an Elastic-stack component) for central management — fine if
  you're all-in on Elastic, heavy if Winlogbeat was just your Windows log shipper.
- The Winlogbeat binary also carries an Elastic-License dependency (`elastic-agent-client`) — relevant if
  you cared about it being Apache-2.0.

## Honest fit check — read this first

**Verified against:** Elastic's published Winlogbeat reference (the current pages, read 2026-10-16; the Winlogbeat version is not recorded). No end-to-end run, and none against Winlogbeat itself.

**Good fit:** you used Winlogbeat as a generic **Windows Event Log shipper** — into Logstash, a syslog
collector, or a non-Elastic SIEM — and want a small static binary with disk buffering and no stack
dependency.

**Not (yet) a fit — Elastic Agent is the saner move if you depend on:**
- **Winlogbeat's ECS field mapping** — the agent can index directly into Elasticsearch (the `elasticsearch`
  output, **Apex** like every non-syslog output) with the core ECS fields (`@timestamp`, `message`,
  `host.name`, `log.syslog.*`), but the Windows event data lands under `labels`, not in Winlogbeat's
  `winlog.*` / `event.code` layout — dashboards and detections built on Winlogbeat data will not match
  without a mapping on the Elastic side. (An `otlp` output is also available.)
- **Kibana dashboards / modules shipped with Beats** — those assume ECS.

**Licensing.** The Windows Event Log input and the disk spool are **Apex** capabilities, as is every
non-syslog output (and UDP syslog). On an unlicensed (Core) agent the Event Log input is removed at load and the spool becomes
drain-only — so this migration needs an Apex entitlement (included at no charge with a licensed logrok
deployment); see [LICENSING](../LICENSING.md).

## Concept mapping

| winlogbeat.yml | Here | Notes |
|---|---|---|
| `winlogbeat.event_logs: - name: X` | `inputs: - type: windows_eventlog` + `channels: [X]` — **Apex** | multiple channels per input, or several named inputs |
| `event_id: 4624,4625` / `level: error` | XPath `query:` (e.g. `*[System[(EventID=4624 or EventID=4625)]]`) or `filter`/`expr` processors on `event_id`/severity fields | XPath filters at the source (cheapest); processors filter in the agent |
| `ignore_older` | n/a — first run starts at **now** (bookmark created), so old backlog isn't re-shipped | bookmarks resume exactly across restarts |
| `processors: drop_event / add_fields` | `processors:` — `filter`, `expr` (conditional set/drop/rename), `add_fields` | |
| `output.logstash` | `outputs: - type: beats` (Beats protocol, TLS/mTLS, failover) — **Apex**; or `type: syslog` (TCP/TLS/mTLS is Core; UDP is Apex) | `beats` keeps Logstash's existing `beats` input on 5044 — no Logstash change; `syslog` points at a Logstash `syslog` input, any SIEM's syslog listener, or logrok |
| `output.elasticsearch` | `outputs: - type: elasticsearch` (bulk API, data stream or index) | core ECS fields; Windows event data under `labels` (see above) — **Apex** |
| `queue.disk` (rarely enabled) | `buffer:` disk spool — CRC-framed, bounded, `when_full` policies, at-least-once | on by config, designed-in rather than bolted-on — **Apex** |
| Fleet / Elastic Agent policies | logrok control plane (agent enrollment, configuration pull, heartbeat) — or fully unmanaged | no stack dependency either way |
| `Microsoft-Windows-Sysmon/Operational` | same channel — see the [Sysmon preset](../../configs/sysmon.example.yaml) | |

## Staged migration through a gateway (no big-bang)

You do not have to replace every Winlogbeat at once. Run one agent as a **gateway** with the `beats_in`
input, point the existing Winlogbeats' `output.logstash.hosts` at it, and send onward with `beats` (to the same
Logstash) or any other output:

```yaml
inputs:
  - { type: beats_in, listen: "0.0.0.0:5044" }
outputs:
  - { type: beats, endpoint: "logstash.example.com:5044" }
```

The Winlogbeats keep running unchanged and their `winlog.*` / `event.code` values pass through as fields;
the gateway adds its disk spool, so a Logstash outage no longer depends on each Beat's own queue. Then replace
the Winlogbeats one host at a time with the agent's own `windows_eventlog` input. Note that events collected
by the agent itself carry its own field names (`event_id`, `provider`, `channel`, EventData keys), not
Winlogbeat's — plan the Logstash or index mapping for those before the cut-over. Both Beats modules are
**Apex**.

## Convert your winlogbeat.yml

Get `luna-migrate` first: a separate download published with every agent release (`luna-migrate-<os>-<arch>`, saved as `luna-migrate` / `luna-migrate.exe`) — see [Getting luna-migrate](../USER-GUIDE.md#getting-luna-migrate).

The `luna-migrate` tool can read a `winlogbeat.yml` and write the equivalent agent configuration, with a report of
everything it could not carry over. Run it on any machine (the file is only read):

```powershell
luna-migrate beats winlogbeat.yml -out agent.yaml -report conversion.json
```

`luna-migrate beats` recognises a Winlogbeat file by its `winlogbeat` section. The output goes to
`-out` (an existing file is never overwritten); `-report` also writes the report as JSON; the
summary always prints. `-target native` (default) converts the Beat's own output to the agent's
equivalent; `-target syslog -syslog-endpoint host:port` sends everything as RFC 5424 syslog
instead. Dotted keys (`winlogbeat.event_logs: …`) and nested keys read the same.

### What is converted

Each `event_logs` entry becomes one `windows_eventlog` input. Event-ID, level and provider filters become one
filter — **AND** between the three, **OR** within each — written as a structured query (`<QueryList>…`) for a
live channel and as an XPath for an `.evtx` file. The table shows the filter part.

| winlogbeat.yml | agent.yaml |
|---|---|
| `name: Security` | `channels: [Security]` (+ `bookmark_path` under `C:\ProgramData\logrok-universal-agent\bookmarks`) |
| `name: C:\Logs\x.evtx` (absolute path) | `file:` — a one-shot drain of that file, no bookmark |
| `id` | names the generated input (`winlog_<id>`) |
| `event_id: 4624, 4700-4702` | `*[System[(EventID=4624 or (EventID>=4700 and EventID<=4702))]]` |
| `event_id: -4735, -4701-4710` (exclusions) | `EventID!=4735 and (EventID<4701 or EventID>4710)`; exclusions alone mean "everything except" |
| `level: error, warning` | `(Level=2 or Level=3)` — critical/crit 1, error/err 2, warning/warn 3, information/info 0 or 4, verbose 5 |
| `provider: [A, B]` | `(Provider[@Name='A'] or Provider[@Name='B'])`; names containing quotes are quoted correctly |
| `xml_query` (needs `id`) | `query` verbatim; channels are read from its `Path=` attributes. Marked `warning`: verify it on a test host |
| `include_xml: true` | `render: xml` (raw event XML as the message; structured fields still attached) |
| `fields`, `tags`, `processors`, `output.*` | converted where the agent has an equivalent; the rest is reported — see the report |

### What is not converted (always in the report)

| Setting | Severity | Why |
|---|---|---|
| `registry_file`, `registry_flush` | info | the agent keeps its own bookmarks; the Beat's registry is not read, so the first run starts at **new** events |
| `shutdown_timeout`, `batch_read_size` | info | not needed: the agent drains on shutdown and sizes its own batches |
| `ignore_older` | info (warning for `.evtx`) | live channels start at new events, so backlog is never read; a file drain reads the whole file |
| `forwarded`, `ignore_missing_channel` | info (`forwarded: false` is a `warning`: there is no equivalent, and ForwardedEvents is subscribed like any channel) | no switch needed; a missing channel is skipped with a warning, the others continue |
| `no_more_events: stop` | warning (live) | a live channel keeps waiting for events; use an `.evtx` file for a one-shot read |
| `language` (non-zero) | warning | messages render in the system language |
| `event_id` / `level` that cannot be parsed | manual | **not** narrowed silently: the whole channel is collected until you write the query |
| options next to `xml_query` that Winlogbeat forbids | warning | ignored; the query decides |
| any unrecognised key | manual | never dropped silently |

Windows accepts at most 20 expressions in a bare XPath query, which is why live channels get the structured
form. For an `.evtx` file (which keeps a bare XPath) a filter above 20 expressions is flagged `warning`.

### Cut-over

1. Convert, then read the report — every `manual` line needs a person; every `warning` a look.
2. Install the agent on a test host with the generated file, check `logrok_agent_events_out_total` climbs, and compare the
   volume per channel with Winlogbeat's.
3. Roll out with Winlogbeat still running (brief double delivery beats a gap), then stop and remove the
   Winlogbeat service.
4. Remember the field names change (`event_id`, `provider`, `channel`, EventData keys instead of `winlog.*` /
   `event.code`): plan the receiving-side mapping first.

## Translation example

```yaml
# winlogbeat.yml (representative)
winlogbeat.event_logs:
  - name: Application
  - name: System
  - name: Security
    event_id: 4624, 4625, 4672
  - name: Microsoft-Windows-Sysmon/Operational
output.logstash:
  hosts: ["logstash.example.com:5044"]
```

Equivalent `agent.yaml` (output goes to a syslog listener — e.g. Logstash `syslog` input on 5514; to keep
Logstash's `beats` input instead, use `- { type: beats, endpoint: "logstash.example.com:5044" }`):

```yaml
service: { mode: service, log_level: info, metrics_listen: "127.0.0.1:9090" }
buffer:  { enabled: true, dir: 'C:\ProgramData\logrok-universal-agent\spool' }
inputs:
  - type: windows_eventlog
    channels: [Application, System, Microsoft-Windows-Sysmon/Operational]
  - type: windows_eventlog
    name: security-auth
    channels: [Security]
    query: "*[System[(EventID=4624 or EventID=4625 or EventID=4672)]]"
outputs:
  - type: syslog
    endpoint: "logstash.example.com:5514"
    tls: true
    ca_file: 'C:\ProgramData\logrok-universal-agent\ca.pem'
```

## Cut-over procedure

1. **Translate** each `event_logs` entry (table above); option reference: [CONFIGURATION.md](../CONFIGURATION.md).
2. **Prepare the receiving side**: a syslog input on Logstash/your SIEM (most have one natively); map the
   RFC 5424 structured-data fields (`event_id`, `provider`, `channel`, EventData keys) into your schema.
3. **Run side-by-side** with an `add_fields` marker (`migration: candidate`); compare event volumes
   against Winlogbeat's.
4. **Mind the position gap:** Beats registry state doesn't transfer; the agent starts at *now*. Start it
   before stopping Winlogbeat and accept brief double-delivery — duplicates beat gaps.
5. **Cut over**: stop/uninstall the Winlogbeat service; remove the marker field.
6. **Verify**: `/metrics` shows `logrok_agent_events_out_total` climbing and `logrok_agent_events_dropped_total` at 0; dashboards on the
   new field names populate.
