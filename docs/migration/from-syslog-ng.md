# Migrating from syslog-ng OSE

This guide is for hosts where syslog-ng Open Source Edition runs as a **forwarder**: it reads local files, the
systemd journal or a local relay listener and sends the messages on — to a central syslog server, Elasticsearch,
Kafka, Grafana Loki or Splunk. The agent can take over that job and keep delivering to the same place, or send
the same data as RFC 5424 syslog to logrok or any syslog receiver.

You don't rewrite the configuration by hand: `luna-migrate` reads `syslog-ng.conf`, follows its `@include`
statements, and writes an agent configuration plus a report of everything it could not carry over.

A syslog-ng **server** (a central collector that receives from many hosts and writes files or databases) is a
different role; the agent does not replace it. If you would rather keep syslog-ng on a host, the agent can also
sit beside it: point a syslog-ng `syslog()` or `network()` destination at the agent's `syslog_in` input.

## Is it a fit?

**Verified against:** syslog-ng OSE 4.x (administration guide read 2026-10-16; syslog-ng 4.12.0, configuration version 4.2). A configuration was validated and run by syslog-ng 4.12.0 and the converted result delivers the same messages. A configuration whose `@version` is newer than 4.12 gets an informational note in the report.

Check these first; they decide whether the conversion is a drop-in or a project.

| If your syslog-ng… | Then |
|---|---|
| reads files (`file()`, `wildcard-file()`), the journal (`system()`, `systemd-journal()`), local sockets (`unix-dgram()`, `unix-stream()`) or listens on the network (`network()`, `syslog()`, `tcp()`, `udp()`, with TLS) | converted (tables below) |
| routes with several `log { }` paths, filters and `flags(final)` / `flags(fallback)` / `flags(catchall)` | converted: the agent routes with a condition per destination (see [Log paths and routing](#log-paths-and-routing)) |
| filters with `facility()`, `level()` / `priority()`, `program()`, `host()`, `message()`, `match()`, `filter()`, `source()` and `and` / `or` / `not` | converted |
| sends to `network()`, `syslog()`, `elasticsearch-http()`, `kafka()`, `loki()` or `splunk-hec-event()` | converted |
| writes **local files** (`file()` destinations) | **not converted**: the agent forwards and does not write log files. Keep local files with the operating system's logging, or keep syslog-ng for local archiving |
| formats messages with **templates**, or rewrites them with `subst()`, macros and template functions | not converted: the agent sends the original message in its own format |
| parses with **patterndb** (`db-parser()`), `regexp-parser()`, `date-parser()` or the application parsers | not converted (JSON, key=value, CSV and XML parsers are) |
| uses `if` / `elif` / `else`, `junction` or `channel` inside log paths | not converted; express the branches as separate log paths before converting, or as `when:` conditions afterwards |
| sends with the generic `http()` destination, `program()`, `pipe()`, a database or a message queue other than Kafka | not converted natively; use `-target syslog` or configure the destination by hand |

## How it fits together

```
before:   files / journal / sockets / relay ──► syslog-ng ──► network | syslog | elasticsearch-http | kafka | loki | splunk-hec-event   (by log path)
after:    files / journal / sockets / relay ──► agent     ──► the same destinations, each with a condition that selects what its log paths delivered
          (or)                                  agent     ──► RFC 5424 syslog over TLS (logrok / any syslog receiver)
```

## Step 1 — convert

Get `luna-migrate` first: a separate download published with every agent release (`luna-migrate-<os>-<arch>`, saved as `luna-migrate`) — see [Getting luna-migrate](../USER-GUIDE.md#getting-luna-migrate).

Run the converter on the syslog-ng host (it only reads the configuration files):

```bash
luna-migrate syslog-ng /etc/syslog-ng/syslog-ng.conf \
  -out /etc/logrok-universal-agent/agent.yaml \
  -report syslog-ng-conversion.json
```

| Flag | Default | Meaning |
|---|---|---|
| `-out` | *(required)* | where to write the agent configuration; an existing file is never overwritten |
| `-report` | — | also write the full report as JSON |
| `-target` | `native` | `native`: send where syslog-ng sent. `syslog`: everything to one RFC 5424 receiver over TLS |
| `-syslog-endpoint` | — | `host:port` of the syslog receiver for `-target syslog` |
| `-syslog-plain` | off | send the syslog output over plain TCP instead of TLS (a receiver without TLS, or a lab) |
| `-start-at` | `end` | where file inputs start on first run: `end` (only new lines: nothing syslog-ng already sent is sent again) or `beginning` |

`@include` is followed the way syslog-ng follows it: a file, every file matching a wildcard in the file name, or
every file of a directory (not recursively, skipping names that start with `~` or `.`, upper-case names first),
up to 15 levels deep. Relative names are looked up beside the including file and beside `syslog-ng.conf`.
`@include "scl.conf"` (syslog-ng's own driver library) is not read — the converter knows the drivers it defines.
`@define` values are substituted wherever they are used (`` `NAME` ``); a backtick name that is not defined in
the configuration is an environment variable, which is never expanded — it is left as written and reported. A
path may name an object defined later or in another file, as in syslog-ng.

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
| `skipped` | deliberately not converted: `internal()` (syslog-ng's own messages) |

Each finding names the file and line it came from. Then fill in what the report lists under "fill in before
deploying". Passwords, HEC tokens and SASL credentials are **never copied** out of the syslog-ng configuration:
the generated file says `REPLACE_WITH_…` (search for it). Key files are referenced by path; a key that needs a
passphrase is not supported — use an unencrypted key file the agent can read.

## Log paths and routing

syslog-ng sends a message through every `log { }` path that lists its source; each path runs its filters,
parsers and rewrites in order and hands what passes to its destinations. The agent has one stream: inputs →
processors → outputs, each output with an optional `when:` condition. The converter keeps the behaviour like
this:

- For every destination it works out the condition under which syslog-ng delivered a message to it: the message
  came from one of the path's sources, passed the filters written **before** the destination in that path, and
  was not taken by an earlier `flags(final)` path. A destination listed by several paths gets the union.
- The source a message came from is carried in the field `syslogng.source` (the syslog-ng source's name), stamped
  by the converted inputs — only when a condition needs it. Because `when:` conditions are evaluated after the
  processors, the field is then delivered with every event (as a structured-data element on syslog, a field
  elsewhere); the report says so.
- `flags(final)`: a message every filter of the path matched is withheld from the later paths that share a
  source. `flags(fallback)`: the path receives what no other path took. `flags(catchall)`: the path takes every
  source. Embedded `log { }` paths receive what passed the outer path.
- A destination that receives everything gets no condition. Routing conditions and more than one destination are
  Apex features.
- A message that matched several paths to the same destination was sent there once per path by syslog-ng; the
  agent sends it once.
- A source whose paths end only in destinations that are not converted is not converted (the report says which).

A filter the agent cannot express (for example `netmask()`, `tags()`, `in-list()`, a value comparison such as
`"${HOST}" eq "web"`, or a regular expression that needs PCRE features) makes that whole filter match everything:
the destination receives **more** than syslog-ng sent it, never less, and the report marks it `manual`.

### Filters

| syslog-ng | Agent condition |
|---|---|
| `facility(auth, authpriv)`, `facility(local0..local7)`, numeric codes | `facility == 4 \|\| facility == 10`, `facility >= 16` |
| `level(warning..emerg)`, `priority(err)`, names or numbers | `severity <= 4`, `severity == 3` |
| `host("regex")` | `host matches "regex"` |
| `message("regex")`, `match("regex" value("MESSAGE"))` | `message matches "regex"` |
| `match("regex" value("name"))` on a name-value pair | the event field of that name (exists only if a parser or the input creates it) |
| `match("regex")` without `value()` | `message matches` — syslog-ng searched the program prefix too; the agent's message holds it only for RFC 3164 messages received over the network (warning) |
| `program("regex")` | the program of RFC 5424 senders (the `app` field), of journal entries (`syslog_identifier`) and the start of an RFC 3164 message received over the network. **Lines read from files carry no program in the agent** — syslog-ng parsed one out of each line — so for file sources the filter does not match (warning) |
| `type(string)` with `flags(prefix)` / `flags(substring)`, `type(glob)`, `flags(ignore-case)` | exact comparison, `startswith`, `contains`, an anchored pattern, `(?i)` |
| `filter(name)`, `source(name)`, `and` / `or` / `not`, parentheses | inlined; `and` binds tighter than `or`, as in syslog-ng |

Regular expressions use the agent's syntax (RE2): most PCRE expressions are the same, but lookarounds and
backreferences are not available — such a filter is reported. In double-quoted syslog-ng strings a backslash is
removed before an ordinary character (`"\d"` is `d`); write `"\\d"` or `'\d'`, as syslog-ng itself requires.

## What is converted

### Sources

| syslog-ng | Agent |
|---|---|
| `file("path")` | `filetail` `path`, with resume state (`state_path`). syslog-ng parsed each line as a syslog message; the agent forwards each line as written, with the facility and severity syslog-ng gave lines without a priority (user / notice) unless `default-facility()` / `default-priority()` say otherwise |
| `wildcard-file(base-dir() filename-pattern() recursive() exclude-pattern())` | `filetail` `path` (`base/pattern`, or `base/**/pattern` with `recursive(yes)`), `exclude` |
| `follow-freq()`, `host-override()`, `encoding("UTF-16LE"/"UTF-16BE")` | `poll_interval`, `host_override`, `encoding` |
| `multi-line-mode(indented)` / `(prefix-garbage)` + `multi-line-prefix()` / `(prefix-suffix)` + `multi-line-suffix()`, `multi-line-timeout()` | `multiline.line_start_pattern` / `line_end_pattern`, `flush_timeout` |
| `system()` | `journald` with its own cursor — what `system()` reads on a Linux host running systemd (on a host without systemd use `syslog_in` with `socket: /dev/log`; reported) |
| `systemd-journal(matches("_SYSTEMD_UNIT" => …, "SYSLOG_IDENTIFIER" => …))` | `journald` `units` / `identifiers`; other match fields are reported (the input reads more) |
| `network()`, `syslog()`, `tcp()`, `udp()` (and the IPv6 variants), `ip()`, `port()`, `transport("tcp"/"udp"/"tls"/"auto")` | `syslog_in` `listen` + `protocols`, or `tls_listen`. Default ports are syslog-ng's: 514 for `network()`, 601 (TCP) / 514 (UDP) / 6514 (TLS) for `syslog()`. RFC 5424 and RFC 3164 are both recognised, and octet-counted or newline framing is detected |
| `tls(cert-file() key-file() ca-file() peer-verify())` on a listener | `tls_cert_file`, `tls_key_file`; client certificates are required (`tls_client_ca_file`) when `peer-verify()` requires them (its default) |
| `unix-dgram()`, `unix-stream()`, `perm()` | `syslog_in` `socket`, `socket_protocol`, `socket_mode` |
| `internal()` | skipped: the agent reports on itself in its own log and metrics |

### Destinations

| syslog-ng | Agent |
|---|---|
| `network("host" port() transport())`, `tcp()`, `udp()` | `syslog` output (`endpoint`, `protocol`, `tls`). syslog-ng sent RFC 3164 here unless `flags(syslog-protocol)`; the agent sends RFC 5424, so the receiver must accept it (a syslog-ng receiver needs a `syslog()` source or `flags(syslog-protocol)`) — warning |
| `syslog("host" …)` | `syslog` output with `framing: octet-counting` over TCP and TLS, as `syslog()` sends |
| `tls(ca-file() cert-file() key-file() peer-verify())` | `ca_file`, `cert_file`, `key_file`; a `peer-verify()` that did not check the server becomes `insecure_skip_verify` (warning) |
| `failover(servers() failback(tcp-probe-interval()))` | `failover_endpoints`, `failback_every` |
| `disk-buffer()` | replaced by the agent's disk spool, which is off until you set `buffer.enabled: true` and `buffer.dir`; the converter does not write a `buffer:` section, so add one |
| `elasticsearch-http(url() index() user() password() tls())` | `elasticsearch` (`endpoints`, `index`, `username`, password placeholder, `ca_file`). Documents are the agent's Elastic Common Schema shape, not syslog-ng's JSON |
| `kafka(bootstrap-servers() topic() key() message() config())` | `kafka` (`brokers`, `topic`, `key_field`, `encoding`, SASL and TLS from `config()`, placeholders for secrets). The record is the agent's JSON event unless the message was just `$MSG` |
| `loki(url() labels() tenant-id() auth(tls()))` | `loki` (`endpoint`, static `labels`, `"host" => "$HOST"` → `label_fields`, `tenant`, `ca_file`). syslog-ng pushes to Loki over gRPC; the agent uses Loki's HTTP API, which listens on another port (3100 by default) — check the endpoint (warning) |
| `splunk-hec-event(url() token() index() source() sourcetype())` | `hec` with a token placeholder. Index, source and sourcetype keep what syslog-ng sent, including its defaults (`main`, `syslog-ng`, `nix:syslog`) |
| any destination, `-target syslog` | one `syslog` output over TLS (`-syslog-endpoint`), with a `when:` that selects what the original destinations received |

### Parsers and rewrites

| syslog-ng | Agent |
|---|---|
| `json-parser(prefix() template())` | `parse_json` (`prefix`, `source`) |
| `kv-parser(prefix() value-separator() pair-separator())` | `parse_kv` (single-character separators; syslog-ng's default pair separator is a comma and/or white space, the agent's a space — warning) |
| `csv-parser(columns() delimiters() prefix() flags(drop-invalid))` | `parse_csv` (one separator character; syslog-ng's default is a space) |
| `xml(prefix())` | `parse_xml` |
| `rewrite { set("text" value("name")); unset(value("name")); }` | `expr` rules, conditioned on the path they were in (its sources and the filters before them) |

The parse processors run on every event; one that syslog-ng ran on only some paths is reported (parsing only
adds fields, and a line that does not parse passes untouched, except where `on_error: drop` is set: `csv-parser()` with `flags(drop-invalid)` drops a non-matching row from every destination, not only the path it was on, and is reported as a warning).

## What is not converted (and what to do)

| syslog-ng | Why / instead |
|---|---|
| `file()`, `pipe()`, `program()`, `usertty()` destinations | the agent does not write local files or run programs; keep these with the operating system's logging or with syslog-ng |
| templates (`template()` statements and options), `template-escape()` | the agent sends the original message in its own format; receivers that parse a custom layout need changing |
| `subst()`, `set()` with macros, `set-tag()`, `set-facility()`, `set-severity()`, `groupset()` and other rewrites | no equivalent in the agent's rules |
| `db-parser()` (patterndb), `regexp-parser()`, `date-parser()`, application parsers | events reach the destinations unparsed; parse downstream |
| `netmask()`, `tags()`, `in-list()`, `rate-limit()`, value comparisons | the filter is treated as matching everything (more is delivered); express it by hand |
| `if` / `elif` / `else`, `junction`, `channel` | write separate log paths, or `when:` conditions |
| `program-override()` | on a `file()` source, a literal value becomes the `app` field, which the syslog output sends as the APP-NAME; on a network source it is not carried over (the program of a received message stays the sender's) |
| `keep-hostname(no)` (syslog-ng's default) | syslog-ng replaced the host name of network messages with the sender's address or name; the agent keeps the host name written in the message (warning) |
| `http()`, `splunk-hec-raw()`, `opentelemetry()` and other destinations | configure by hand, or use `-target syslog` |
| `default-network-drivers()`, `pipe()`, `program()` and other sources | add a `syslog_in` input (or the input that fits) by hand |
| `ca-dir()` | the agent takes one CA file; concatenate the directory's certificates into one PEM file |
| syslog-ng's persist file (read positions) | the agent keeps its own state and does not import syslog-ng's positions |

## Step 3 — cut over without a gap

Prefer a few seconds of duplicates to a gap:

1. Install the agent with the converted configuration and start it. With `start_at: end` file inputs read only
   lines written from now on.
2. Confirm in the destination that events arrive from the agent.
3. Stop and disable syslog-ng.

Between steps 1 and 3 both send new file lines, so those few seconds arrive twice. Stopping syslog-ng *first*
would lose the lines written before the agent starts.

**Listeners and sockets are the exception.** A network port or a local socket such as `/dev/log` can be opened by
one program only, so the agent cannot listen while syslog-ng still does. For those inputs, stop syslog-ng
immediately before starting the agent (messages sent during the switch can be lost), or let the agent listen on
a new port and move the senders over.

The events differ in shape from syslog-ng's output: the syslog output sends RFC 5424 with the event's `app`
field as APP-NAME and its process id as PROCID — when there is no `app` field, the event's source
(`file:<path>`, `journald:<unit>`, `syslog-in`) is used instead, and for a message received from an RFC 5424
sender, or a file source with `program-override()`, that is the sender's or the overridden program. A source that
differs from the APP-NAME also rides in full as a `source` structured-data parameter, along with the event's other
fields; the Elasticsearch, Kafka and Loki outputs send the agent's own document or
record shape. Dashboards and parsers that read syslog-ng's template output need a look.

## Sending to logrok instead

```bash
luna-migrate syslog-ng /etc/syslog-ng/syslog-ng.conf -out agent.yaml \
  -target syslog -syslog-endpoint logrok.example.com:16575
```

Sources, filters and processors convert as above; only the destination changes, and the `when:` keeps only what
syslog-ng's destinations received. Add the TLS trust (`ca_file`, and a client certificate if your receiver
requires one) to the generated `syslog` output.

## Licensing

The converter is free. What the converted configuration uses follows the normal rules
([LICENSING](../LICENSING.md)): file tailing, journald, the syslog input, the parse and `expr` processors and one
RFC 5424 syslog destination over TCP/TLS are Core; the `elasticsearch`, `kafka`, `loki` and `hec` outputs, UDP
sending, more than one destination and per-destination routing conditions (the `when:` that log-path routing
becomes) and the disk spool are Apex.
