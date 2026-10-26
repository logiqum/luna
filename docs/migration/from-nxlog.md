# Migrating from NXLog (CE / Enterprise) to logrok-universal-agent

## Why you're probably reading this

- **NXLog Enterprise Edition + NXLog Manager: End-of-Sale since 2024-09-19** — no new licenses; the
  vendor's path is NXLog Platform. ([announcement](https://nxlog.co/news-and-blog/posts/announcing-eos-nxlog-ee-manager))
- **NXLog Community Edition is in maintenance mode** — "occasional maintenance updates," no new
  features. ([NXLog's own comparison](https://nxlog.co/news-and-blog/posts/nxp-vs-ce))
- NXLog Platform is priced **per source** ($5.58–$12.12/source/month at published tiers,
  [pricing](https://nxlog.co/platform/plans)).

If you run NXLog agents primarily to ship **Windows Event Log + files as syslog to a SIEM**, this agent
covers that path with a single static binary with no runtime dependencies, disk store-and-forward, and TLS/mTLS.

## Honest fit check — read this first

**Verified against:** NXLog Enterprise Edition 5.8 reference manual and NXLog Community Edition 3.2 manual. Delivery into syslog-ng was run end to end; the converter was not run against NXLog itself.

**Good fit:** Windows Event Log (any channel incl. Security and Sysmon) and/or file tailing, forwarded
as RFC 5424 syslog over TCP/TLS or UDP to any aggregator; air-gapped or flaky-link sites;
Linux journald/syslog relays.

**Not (yet) a fit — stay on NXLog or wait for our roadmap if you depend on:**
- **FIM** (`im_fim` — an NXLog Enterprise feature). Not a like-for-like replacement, and worth being precise about, because it is usually the first thing an Enterprise user checks. The agent forwards file-change **events** — auditd watch rules on Linux, Sysmon or native object-access auditing on Windows — with the acting user and command line attached. It does **not** baseline files or alert on unauthorised change the way `im_fim` does; that comparison belongs in the platform the events land in (the `file_hash` processor — Apex — can attach the changed file's content digest so the platform has something to compare). If what you need from `im_fim` is the evidence stream, it is here today; if you need the endpoint itself to decide what changed, it is not. See [USER-GUIDE § File-integrity events](../USER-GUIDE.md#file-integrity-events-fim--what-the-agent-does-and-what-it-deliberately-doesnt). ETW (`im_etw`) is now shipped as the `etw` input — see the parity table below.
- **Exotic platforms beyond AIX/Solaris/BSDs** (HP-UX — not supported — and the deeper 100+-OS NXLog tail). **AIX (ppc64) and Solaris (amd64) binaries ship as an extended-tier platform** — **Solaris (amd64) is now runtime-verified on the real OS** (as a hypervisor guest, not bare metal); **AIX (ppc64) runs on a real AIX 7.3 server in every release check.** We are looking for design partners with AIX/Solaris estates (banks on Power: this means you)
- **ODBC / direct-to-database output** — not planned; write to your aggregator and let it land the data.
  Other non-syslog sinks are shipped, not future: `otlp` (gRPC + HTTP), `kafka`, `s3`, `sentinel`, `xsiam`,
  `loki`, `hec` (Splunk HTTP Event Collector), `snare`, `elasticsearch`, `beats` and `fluent`. All of them are **Apex**, as is UDP (data-diode) egress; the Core output is
  syslog over TCP/TLS, which most SIEMs ingest natively
- **NXLog's procedural config language** (`Exec` blocks with full rewriting) — our `expr` processor covers
  conditional set/drop/rename; heavier transformation belongs in your aggregator

## Concept mapping

| NXLog | Here | Notes |
|---|---|---|
| `<Input>` + `Module im_msvistalog` | `inputs: - type: windows_eventlog` | multi-channel; XPath `query` supported — **Apex** |
| `QueryXML` / `Query` | `query:` (XPath) or `channels:` list | same Windows query syntax |
| `SavePos TRUE` | automatic (bookmarks at `bookmark_path`) | **positions do not transfer** — see cut-over below |
| `Module im_file` + wildcards | `type: filetail` (`path`/`paths`, `**` globs, `exclude`) | rotation + multiline + restart-resume built in |
| `xm_multiline` | `multiline:` block on `filetail` (start/end pattern + flush timeout) | |
| `Module im_systemd` | `type: journald` (Linux) | unit/priority filters, cursor resume |
| `Module im_tcp`/`im_udp` (+syslog parsing) | `type: syslog_in` | RFC 5424/3164 auto-detect, SD→fields |
| `om_tcp`/`om_ssl` + `to_syslog_ietf()` | `outputs: - type: syslog` (RFC 5424; `tls`, mTLS) | newline or RFC 6587 octet-counting framing |
| `om_udp` + `to_syslog_*()` | `protocol: udp` (one datagram/event; optional `sequence` SD) | data-diode compatible — **Apex** (Core is TCP/TLS) |
| `Exec` drop/rewrite logic | `processors:` — `filter`, `expr`, `add_fields` | conditions on severity/fields/message |
| `xm_json` parsing | ✅ `processors: - type: parse_json` (message → fields, nested flatten with dots, number literals preserved; non-JSON passes through — safe on mixed streams) | `prefix`/`source`/`drop_message`/`on_error` options |
| `xm_csv` parsing | ✅ `processors: - type: parse_csv` (positional columns, RFC 4180 quoting, W3C/IIS via space separator + `-` placeholders) | `columns`/`separator`/`prefix`/`source`/`on_error` |
| `xm_xml` parsing | ✅ `processors: - type: parse_xml` (XML message → fields; non-XML passes through) | or forward raw and parse in the aggregator (logrok/syslog-ng parse natively) |
| `im_uds` (`/dev/log`) | ✅ `syslog_in` with `socket: /run/logrok/log.sock` (unixgram default, stream optional, mode control) | refuses to replace non-socket files; stale sockets cleaned up |
| WEF collection (`im_wseventing`, EE) | point `windows_eventlog` at the WEC's `ForwardedEvents` channel — **works today** | acting as a WEC *server* is deliberately out: an agent per endpoint replaces WEF at scale |
| `im_wmi` (EE) | `inputs: - type: wmi` — **Apex** | WMI/CIM polling queries |
| `im_etw` (EE) | `inputs: - type: etw` — **Apex** | real-time ETW trace session; decodes events from manifest-based providers |
| NXLog Manager / Platform agent mgmt | the **logrok control plane** — or **any implementation of the documented enroll/config/heartbeat contract**; enroll/config-pull/heartbeat are shipped and verified end-to-end on both sides | **the agent also runs fully unmanaged** — central management is optional, and the protocol is documented rather than proprietary, so managing it is not tied to one vendor |

## Convert your nxlog.conf

Get `luna-migrate` first: a separate download published with every agent release (`luna-migrate-<os>-<arch>`, saved as `luna-migrate` / `luna-migrate.exe`) — see [Getting luna-migrate](../USER-GUIDE.md#getting-luna-migrate).

The agent can read an NXLog configuration (Community or Enterprise `nxlog.conf`, `define`s and `include`
files followed) and write the equivalent agent configuration plus a conversion report:

```
luna-migrate nxlog /etc/nxlog/nxlog.conf \
  -out /etc/logrok-universal-agent/agent.yaml \
  -report nxlog-conversion.json
```

On Windows: `luna-migrate.exe nxlog "C:\Program Files\nxlog\conf\nxlog.conf" -out ...`.
The output goes to `-out` (an existing file is never overwritten); the report summary always prints and
`-report` also writes it as JSON. `-target native` (default) maps NXLog's own outputs;
`-target syslog -syslog-endpoint host:port` sends everything as RFC 5424 syslog to one receiver
instead. `-start-at end` (default) starts file inputs at new data, so nothing NXLog already sent is sent
again; `beginning` reads existing content. luna-migrate checks the configuration's structure, then runs the installed agent's `-check-config` (found beside `luna-migrate` or on the `PATH`, or named with `-agent`), which builds every module exactly as that agent version would; a configuration the agent rejects is removed. Without an agent to hand it says so — then run `logrok-universal-agent -check-config -config agent.yaml` on the target host. The converter reads configuration only — it never runs `include_stdout` commands and
never copies a key passphrase or password (placeholders `REPLACE_WITH_...` are listed under "fill in before
deploying").

What converts, and what the report tells you to do by hand:

| NXLog | Result |
|---|---|
| `im_file` (`File` wildcards, several `File` lines, `Recursive`, `Exclude`, `SavePos`, `ReadFromLast`) | `filetail`; `Recursive` becomes a `**` path segment; an `xm_multiline` `HeaderLine` regex becomes `multiline.line_start_pattern` |
| `im_msvistalog` (`Channel`, `Query`, `QueryXML`, `File`) | `windows_eventlog`; a query naming several channels becomes one input per channel, each with its own part of the query |
| `im_etw` (`Provider`, `Level`, `MatchAnyKeyword`) | `etw` |
| `im_tcp`, `im_udp`, `im_ssl`, `im_uds` | `syslog_in` (TLS and client-certificate verification for `im_ssl`) |
| `im_systemd` | `journald` |
| `om_tcp`, `om_ssl`, `om_udp` with `to_syslog_ietf()` | `syslog` output (RFC 5424; failover hosts, TLS, CA and client certificate); `to_syslog_snare()` becomes the `snare` output |
| `om_elasticsearch`, `om_kafka` | `elasticsearch`, `kafka` (document / record shape differs: the report says how) |
| `<Route>` `Path` | each output receives only the inputs routed to it |
| `Exec drop();` | the input is not converted |
| `Exec if $raw_event =~ /re/ drop();` (also `!~`, `==`, `!=`; Event Log fields `$EventID`, `$Channel`, `$SourceName`, `$Message`, `$Hostname`) | a drop rule that applies to that input only |
| `Exec parse_syslog();` on a syslog input; `parse_json();` when every input uses it | handled by `syslog_in` itself / the `parse_json` processor |

Everything else is listed in the report as **manual** with the original line: other `Exec` statements, processor
modules in a route, `om_http`, `om_file` and other outputs, FIM, WMI and other Enterprise inputs, and any
directive the converter does not know. Nothing is skipped silently.

Two things to know about routes. When one input feeds some outputs but not others, the agent marks each event
with a field `nxlog_input` (the NXLog input name) and the outputs filter on it; that field is delivered with the
event, so strip it at the receiver if you do not want it. Routing to several outputs is an **Apex** capability,
like fan-out. A relative `include` is resolved against the including file's directory.

## Translation example

NXLog CE (representative config — adjust to yours):

```
<Extension syslog>  Module xm_syslog  </Extension>
<Input eventlog>
    Module   im_msvistalog
    <QueryXML>
      <QueryList><Query Id="0">
        <Select Path="Application">*</Select>
        <Select Path="System">*</Select>
        <Select Path="Security">*[System[(Level=1 or Level=2 or Level=3)]]</Select>
      </Query></QueryList>
    </QueryXML>
    SavePos  TRUE
</Input>
<Output siem>
    Module   om_ssl
    Host     siem.example.com
    Port     6514
    CAFile   %CERTDIR%/ca.pem
    Exec     to_syslog_ietf();
</Output>
<Route r> Path eventlog => siem </Route>
```

Equivalent `agent.yaml`:

```yaml
# Needs an Apex entitlement: the Event Log input and the disk spool are both Apex.
service: { mode: service, log_level: info, metrics_listen: 127.0.0.1:9090 }
buffer:  { enabled: true, dir: 'C:\ProgramData\logrok-universal-agent\spool' }
inputs:
  - type: windows_eventlog
    channels: [Application, System]
  - type: windows_eventlog
    name: security-errors
    channels: [Security]
    query: "*[System[(Level=1 or Level=2 or Level=3)]]"
outputs:
  - type: syslog
    endpoint: "siem.example.com:6514"
    tls: true
    ca_file: 'C:\ProgramData\logrok-universal-agent\ca.pem'
```

Differences worth knowing: at-least-once delivery, with backpressure available (`buffer.when_full: block`;
the default `drop_oldest` counts every drop in `events_dropped_total`, so nothing is silent), and
Prometheus self-metrics (`service.metrics_listen`) in the free Core tier. Disk store-and-forward
(`buffer:`) is **Apex** — as is NXLog's own buffering beyond its Community limits — and is included at
no charge with a licensed logrok deployment.

## Cut-over procedure

1. **Inventory** your `nxlog.conf` routes: which inputs feed which outputs, and any `Exec` filtering.
2. **Translate** with `luna-migrate nxlog` (above) or by hand per the table; full option reference in [CONFIGURATION.md](../CONFIGURATION.md).
3. **Run side-by-side** for a day: keep NXLog running; start this agent with
   `processors: [{ type: add_fields, fields: { migration: candidate } }]` so the SIEM can distinguish
   the two streams. Compare volumes.
4. **Mind the position gap:** bookmarks/`SavePos` don't transfer. On first start the agent collects
   **from now** (Event Log) / from `start_at` (files). Start the agent *before* stopping NXLog and accept
   a short double-delivery overlap — duplicates are cheaper than a gap.
5. **Cut over:** stop + uninstall the NXLog service, remove the `add_fields` marker. If your SIEM ingests
   NXLog's legacy Snare/MSWinEventLog format, point the agent at it with the `snare` output instead of
   `syslog` — an existing Snare-configured SIEM feed keeps working unchanged during (and after) migration.
6. **Verify:** `curl -s localhost:9090/metrics | grep events_out` climbs; SIEM volume matches the NXLog
   baseline; `events_dropped_total` stays 0.

Compliance users: the [compliance mapping pack](../compliance/README.md) documents how the agent covers
PCI DSS 10 / NERC CIP-007 R4 / NIS2 / DORA / CMMC logging duties.
