# Migrating from rsyslog

rsyslog is the system logger on most Linux distributions. Before you replace it, decide which job it does
on the host. There are two cases.

- **It collects this host's logs** (local programs, the journal, application log files) and forwards them to a
  central receiver. The agent can take that over. It reads the same sources, forwards with disk
  store-and-forward and TLS, and is managed centrally. `luna-migrate rsyslog` converts the configuration.
- **It is a relay**: other machines (network devices, appliances, servers) send to it over UDP/TCP and it
  forwards. Many rsyslog deployments work this way. **You often don't need a migration here.** Keep
  rsyslog and add one forwarding action that points at the agent's `syslog_in` listener (below). The agent
  then adds buffering, TLS and central management in front of your aggregator, and rsyslog's filters keep
  working unchanged. You can also convert the relay itself; listeners, rulesets and filters convert as
  described below.

### Keep rsyslog, point it at the agent

On the agent:

```yaml
inputs:
  - type: syslog_in
    listen: 127.0.0.1:5514
    protocols: [tcp]
```

On rsyslog, add an action. Use RFC 5424 framing so the structured header survives:

```
action(type="omfwd" target="127.0.0.1" port="5514" protocol="tcp"
       template="RSYSLOG_SyslogProtocol23Format")
```

## Is it a fit?

**Verified against:** rsyslog 8 (the daily stable documentation, read 2026-10-16). A configuration was validated by rsyslogd 8.2604.0 and converted.

| If your rsyslog… | Then |
|---|---|
| tails files with `imfile` (RainerScript `input()` or legacy `$InputFileName` … `$InputRunFileMonitor`) | converted to `filetail`, keeping each file's tag, facility and severity |
| reads the journal (`imjournal`) or the local log socket (`imuxsock`) | converted to the `journald` input (see [The local log socket](#the-local-log-socket)) |
| listens with `imudp`, `imtcp` or `imptcp`, including TLS listeners | converted to `syslog_in` |
| forwards with `omfwd` or the `@host` / `@@host` forms, including TLS | converted to the `syslog` output |
| writes to Elasticsearch (`omelasticsearch`) or Kafka (`omkafka`) | converted to the `elasticsearch` / `kafka` outputs; credentials become placeholders |
| filters with selectors (`*.info;mail.none`), property filters (`:msg, contains, "x"`), `if … then` on message, host, program name, severity or facility, and `stop` | converted into conditions on what each destination receives |
| binds inputs to rulesets, uses `call` | converted; each ruleset's events go only to that ruleset's destinations |
| receives or sends RELP (`imrelp`, `omrelp`), posts with `omhttp`, writes to databases or other sinks | **not converted**; reported with what to do instead |
| rewrites messages: templates, `set` variables, `mm*` message modification modules, `foreach`, lookup tables | **not converted**; reported line by line |
| writes local log files (`/var/log/…`, `omfile`) | **not converted**: the agent writes no local log files. Keep rsyslog (or journald) for local files if you need them |
| uses the YAML configuration format | **not converted**; convert the RainerScript form |

## Step 1 — convert

Get `luna-migrate` first: a separate download published with every agent release (`luna-migrate-<os>-<arch>`,
saved as `luna-migrate`) — see [Getting luna-migrate](../USER-GUIDE.md#getting-luna-migrate).

Run it on the rsyslog host (it only reads the configuration files):

```bash
luna-migrate rsyslog /etc/rsyslog.conf \
  -out /etc/logrok-universal-agent/agent.yaml \
  -report rsyslog-conversion.json
```

| Flag | Default | Meaning |
|---|---|---|
| `-out` | *(required)* | where to write the agent configuration; an existing file is never overwritten |
| `-report` | — | also write the full report as JSON |
| `-target` | `native` | `native`: send where rsyslog sent. `syslog`: everything rsyslog forwarded over the network goes to one RFC 5424 receiver over TLS |
| `-syslog-endpoint` | — | `host:port` of the syslog receiver for `-target syslog` |
| `-syslog-plain` | off | send the syslog output over plain TCP instead of TLS (a receiver without TLS, or a lab) |
| `-start-at` | `end` | where file inputs start on first run: `end` (only new lines) or `beginning` |

The converter reads all three syntaxes, which real files mix freely: RainerScript (`module()`, `input()`,
`action()`, `ruleset()`, `if … then … else`, `call`, `stop`), legacy `$` directives, and traditional selector
lines including `&` continuations and `~`. It follows `$IncludeConfig` (a file, a glob or a directory) and
`include()` (`file=` with globs, `mode="optional"`, inline `text=`). Included files are read in name order.
A relative include is resolved against the including file's directory.

luna-migrate checks the configuration's structure, then runs the installed agent's `-check-config` (found
beside `luna-migrate` or on the `PATH`, or named with `-agent`), which builds every module and compiles every
condition. A configuration the agent rejects is removed.

## Step 2 — read the report

| Severity | Meaning |
|---|---|
| `manual` | **not carried over**: a person must decide (each line says what to do instead) |
| `warning` | carried over, with a difference to check |
| `info` | carried over differently, nothing to do |
| `skipped` | deliberately not converted (local files, terminal messages, rsyslog's own marks and statistics) |

Each finding names the file and line it came from. Passwords (`pwd=`, a Kafka `sasl.password`) are **never
copied**: the generated file says `REPLACE_WITH_…` and the report lists every value to fill in. Backtick
constants (`` `echo $VAR` ``, `` `cat file` ``) are never evaluated; every use is reported.

## How filters and rulesets carry over

In rsyslog, every input feeds a ruleset (the default one unless the input names another). Each action runs
when the filters on its path match, unless an earlier `stop` already discarded the message. The agent's
processors and destinations apply to all inputs, so the converter works out, for each destination, the exact
condition under which rsyslog would have reached it:

- **one destination**: the condition becomes a single drop rule (an `expr` processor). An event is forwarded
  exactly when rsyslog would have forwarded it;
- **several destinations**: each gets a `when:` condition. Several destinations and `when:` routing are an
  **Apex** capability;
- **rulesets bound to different inputs**: each input stamps a field `rsyslog_ruleset` with its ruleset's name,
  and the conditions test it. With several destinations, that field is delivered with the event, so strip it at
  the receiver if it is unwanted. With one destination it is removed before sending;
- two actions that send to the same place become one destination that receives what either sent.

Selector semantics are kept: a priority selects that priority and more severe ones, `=` exactly one, `!`
excludes, `none` excludes the facility, and later selectors on a line refine earlier ones.

**A filter the converter cannot express never loses events.** It is reported as `manual`, and the
destination receives everything that filter would have excluded. A `stop` under such a filter is not
applied. You get more than before, never less, until you add the condition by hand. Typical cases:
`$fromhost-ip` and other properties the agent does not keep, variables (`$!…`, `$.…`), functions other than
`re_match`, and POSIX basic regular expressions (`regex`; `ereregex` and `re_match` are converted).

### Program names

`$programname` conditions test the agent's `app` field (set from each `imfile` Tag, and parsed from the
APP-NAME of RFC 5424 messages received over the network) and the journal's `syslog_identifier`. An RFC 3164
message received over the network keeps its tag inside the message text, so a program-name condition does
not match it, unless you set `parse_tag: true` on the `syslog_in` input (the tag then becomes the `app`
field). Check the routing of such senders.

## What is converted

### Inputs

| rsyslog | Agent |
|---|---|
| `imfile` `File` (wildcards) | `filetail` `paths`; resume state kept by the agent |
| `Tag` | the `app` field, which the syslog output sends as the syslog header's APP-NAME (as rsyslog did); the file source (`file:<path>`) travels as structured data |
| `Facility`, `Severity` (defaults local0 / notice) | `facility`, `severity`, always written out |
| `startmsg.regex`, `endmsg.regex`, `readMode="2"` | `multiline.line_start_pattern` / `line_end_pattern` (`readMode` 2 → a line that does not start with a blank starts a record) |
| `imjournal` | `journald` with its own `cursor_file` |
| `imuxsock` system socket | `journald` (see below); an additional `Socket` / `$AddUnixListenSocket` → `syslog_in` `socket` |
| `imudp`, `imtcp`, `imptcp` (`Port` lists, `Address`, legacy `$UDPServerRun` / `$InputTCPServerRun`) | `syslog_in` `listen`; UDP and TCP on the same port become one input with both protocols |
| `imtcp` with `StreamDriver.Mode="1"` | `syslog_in` `tls_listen` with the certificate and key from `DefaultNetstreamDriverCertFile` / `KeyFile`; with `AuthMode` `x509/…` the CA becomes `tls_client_ca_file` |
| `ruleset=` on an input, legacy `…BindRuleset`, `$RuleSet` | routing as described above |

### Destinations

| rsyslog | Agent |
|---|---|
| `omfwd` `target`, `port` (default 514), `protocol` (default UDP) | `syslog` output `endpoint`, `protocol`; a target list becomes `failover_endpoints` |
| `@host[:port]`, `@@host[:port]`, `@@(o)…`, `[IPv6]:port` | `syslog` output over UDP / TCP; `(o)` → `framing: octet-counting` |
| `StreamDriverMode="1"` or `$ActionSendStreamDriverMode 1` | `tls: true`; `DefaultNetstreamDriverCAFile` → `ca_file`, client certificate and key → `cert_file` / `key_file`; with `x509/name` a single permitted peer → `server_name` |
| `TCP_Framing="octet-counted"` | `framing: octet-counting` |
| `omelasticsearch` `server`, `serverport`, `usehttps`, `searchIndex`, `uid` / `pwd`, `tls.cacert`, `allowUnsignedCerts` | `elasticsearch` `endpoints`, `index` (rsyslog's default `system`), `username` / password placeholder, `ca_file`, `insecure_skip_verify` |
| `omkafka` `broker`, `topic`, `confParam` (`security.protocol`, `sasl.*`, `ssl.*.location`, `compression.codec`) | `kafka` `brokers`, `topic`, `tls`, `sasl` (password as a placeholder), `ca_file` / `cert_file` / `key_file`, `compression` |
| action and main queues (`queue.*`, `$ActionQueue…`) | not needed: the agent spools every destination to disk |

**The wire format changes.** rsyslog forwards RFC 3164 lines unless a template says otherwise. The agent's
`syslog` output sends RFC 5424, with the event's fields as structured data. Check that your receiver
parses RFC 5424. Elasticsearch documents and Kafka records take the agent's shape (the report says how).

### The local log socket

rsyslog's `imuxsock` reads what local programs send through `syslog(3)` to `/dev/log`. On a systemd host
those messages are also in the journal, so the converter reads the journal (`journald` input) instead of
taking over the socket. When `imjournal` is configured too, the journal is read once. On a host without
systemd, replace the `journald` input by hand with a `syslog_in` that has `socket: /dev/log`, and stop rsyslog
first: the agent creates that socket itself and replaces one that already exists. Kernel messages
(`imklog`) are in the journal on systemd hosts as well.

## What is not converted (and what to do)

| rsyslog | Why / instead |
|---|---|
| local files, `omfile`, `:omusrmsg:`, pipes, `omprog` | the agent forwards; keep rsyslog or journald for local files |
| `imrelp` / `omrelp` | the agent does not speak RELP; use RFC 5424 over TCP/TLS (`syslog_in` / the `syslog` output) |
| `omhttp` | no generic HTTP output; use `-target syslog`, or one of the HTTP destinations (`hec`, `otlp`, `loki`, `elasticsearch`, `sentinel`, `xsiam`) by hand |
| templates (`template()`, `$template`, `template=` on an action) | the agent sends RFC 5424 (or a JSON body with `encoding: json`) |
| `set` / `unset` / `foreach`, message-modification modules (`mmjsonparse`, `mmnormalize`, …), lookup tables | use the agent's processors (`parse_json`, `parse_kv`, `redact`, `expr`, `lookup`) where they fit |
| `$fromhost-ip`, `$inputname` and other properties the agent does not keep; POSIX `regex` | over-approximated as described above; rewrite with `ereregex` or by hand |
| `$AllowedSender` | restrict the listener with a firewall, or require client certificates (`tls_client_ca_file`) |
| `PermittedPeer` on a TLS listener | the agent accepts every client certificate the CA signed |
| rsyslog's read positions (`$WorkDirectory`, `stateFile`, the journal state file) | not imported; see the cut-over below |

## Step 3 — cut over

Prefer a few seconds of duplicates to a gap:

1. Install the agent with the converted configuration and start it. File inputs start at the end of each file
   (`-start-at end`), and the journal at new entries.
2. Confirm at the receiver that events arrive from the agent with the expected host, facility and severity.
3. Remove rsyslog's forwarding actions (or stop rsyslog, if it has nothing else to do).

**Listeners and sockets cannot run in parallel.** The agent cannot listen on a port rsyslog still holds, and
it takes over a local socket. For those inputs, stop rsyslog's listener first, or convert, change the port for
a trial run, and switch the senders over.

## Sending to logrok instead

```bash
luna-migrate rsyslog /etc/rsyslog.conf -out agent.yaml \
  -target syslog -syslog-endpoint logrok.example.com:16575
```

Every network destination rsyslog used (converted or not) is replaced by one RFC 5424 output over TLS. The
union of their filters decides what is sent. If rsyslog forwarded nothing over the network, every converted
input is sent. Add the TLS trust (`ca_file`, and a client certificate if your receiver requires one) to the
generated `syslog` output.

## Licensing

The converter is free. What the converted configuration uses follows the normal rules
([LICENSING](../LICENSING.md)): file tailing, journald, the syslog input, the `expr` processor and one RFC 5424
syslog destination over TCP/TLS are Core. UDP forwarding, more than one destination (and `when:` routing),
the `elasticsearch` and `kafka` outputs, and the disk spool are Apex.
