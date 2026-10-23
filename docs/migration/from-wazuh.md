# Migrating log collection from the Wazuh (or OSSEC) agent to logrok-universal-agent

## Read this first: what this migration is, and is not

The Wazuh agent does several jobs. This agent does one of them: **it collects logs and forwards them as
standard RFC 5424 syslog** to a receiver you choose. So this guide moves the *log collection* part of a
Wazuh agent — the `<localfile>` blocks — and nothing else.

| The Wazuh agent does | After the migration |
|---|---|
| Collects logs (`<localfile>`: files, Windows event channels, journald, macOS unified log) | **Moves** to this agent |
| Sends events to the Wazuh manager over **Wazuh's own agent protocol** (`<client><server>`, default port 1514) | **Does not move.** The converted agent sends syslog to the receiver you name. The Wazuh manager stops receiving this host's events, and with them everything the manager did: decoding, rules, alerts, agent management |
| File integrity monitoring (`<syscheck>`), rootkit and policy checks (`<rootcheck>`), configuration assessment (`<sca>`), system inventory (`<wodle name="syscollector">`), active response, vulnerability detection and the other modules | **Do not move.** Each one found in your file is listed in the conversion report as `skipped` |
| Runs commands and collects their output (`command`, `full_command`) | **Does not move** — see below |

If you rely on the Wazuh manager's analysis, keep the Wazuh agent. Run this agent beside it (it only reads
the same log files) if you want the same logs to reach a second destination.

The converter never writes a Wazuh manager address into the new configuration and never copies an
enrollment password, agent key or certificate.

## Honest fit check

**Verified against:** the current Wazuh documentation (read 2026-10-16; its pages state no version). A configuration was checked by Wazuh agent 4.14.1 and delivery into syslog-ng was run end to end.

**Good fit:** you want the logs a Wazuh agent tails — application and system log files, Windows Event Log
channels, journald — delivered as syslog over TCP/TLS to a SIEM or log platform, with disk
store-and-forward, in one static binary.

**Not a fit — stay on the Wazuh agent — if you depend on:**
- the Wazuh manager's rules, alerts or dashboards for this host;
- file integrity monitoring, rootkit checks, configuration assessment, inventory or active response from
  the endpoint itself;
- `command` / `full_command` sources (periodic command output) as a primary data source. The agent has no
  input that runs a command; schedule the command with the operating system and tail the file it writes.

## Concept mapping

| Wazuh | Here | Notes |
|---|---|---|
| `<localfile>` `log_format` `syslog`, `snort-full`, `snort-fast`, `apache`, `squid`, `generic`, `mysql_log`, `mssql_log`, `postgresql_log`, `nmapg`, `djb-multilog`, `iis`, `ossecalert`, `json` + `location` | `type: filetail` | In Wazuh the format names the decoder the *manager* applies. Here the line is forwarded unchanged and the receiver parses it |
| `location` wildcards (`*`, `?`) | `paths:` globs | `**` recursion is available here too |
| `location` with a date pattern (`app-%Y-%m-%d.log`) | a glob (`app-????-??-??.log`) | Wazuh resolves the date at run time to one file; the glob follows every dated file. Reported as a warning |
| `exclude` | `exclude:` | |
| `only-future-events` (default `yes`) | `start_at: end` | with `no`, Wazuh also read what was written while it was stopped; `state_path` gives the agent that resume. See cut-over |
| `age` | not converted | reported; with `start_at: end` an old file adds nothing anyway |
| `log_format` `multi-line-regex` + `multiline_regex` (`match` `start` / `end`, `timeout`) | `multiline:` (`line_start_pattern` / `line_end_pattern`, `flush_timeout`) | `match="all"` has no equivalent; `replace` (newline removal) is not done. Both reported |
| `log_format` `multi-line:N` (fixed line count) | not converted | the agent starts a new event at a matching line, it cannot count lines. The file is forwarded line by line and the report says so |
| `log_format` `audit` | `type: linux_audit` | reassembles the records of one audit event (**Apex**) |
| `log_format` `eventchannel`, `location` = channel, `query` (XPath) | `type: windows_eventlog`, `channels:`, `query:` | the XPath is passed through as a structured query on the channel (**Apex**) |
| `log_format` `eventlog` | `type: windows_eventlog` | |
| `log_format` `journald` with `<filter>` | `type: journald` with `units`, `identifiers`, `priority` | only a plain `^unit$`, `^identifier$` and a `[0-N]` priority range convert; any other filter is reported and its entries are forwarded |
| `log_format` `macos`, `<query>` (predicate, `level`) | `type: oslog`, `predicate`, `level` | macOS only (**Apex**); the `type` selector has no equivalent |
| `<label key="k">v</label>` on a `json` source | the field `k` on every event of that input | Wazuh adds the label inside the JSON event; here it is a separate field (a syslog structured-data parameter) |
| `<labels>` (agent labels) | the same fields on every input | keys starting with `_` are reserved by Wazuh and are not carried |
| `<ignore>` / `<restrict>` (`type` `PCRE2` — the default —, `osregex`, `osmatch`) | a drop rule scoped to that input | matched against the message, searched anywhere in the line. See "Filters" below |
| `<client><server>` (address, port, protocol) | **not migrated** | replaced by the `syslog` output (RFC 5424 over TCP with TLS) |

## Convert your ossec.conf

Get `luna-migrate` first: a separate download published with every agent release (`luna-migrate-<os>-<arch>`, saved as `luna-migrate` / `luna-migrate.exe`) — see [Getting luna-migrate](../USER-GUIDE.md#getting-luna-migrate).

Run the Wazuh agent's own check first, so you convert a file Wazuh itself accepts. It ships in the agent
package (checked on 4.14.1) and prints nothing when the file is valid:

```
/var/ossec/bin/wazuh-logcollector -t
```

Then convert. The source is the agent's `ossec.conf` (Linux `/var/ossec/etc/ossec.conf`; Windows
`C:\Program Files (x86)\ossec-agent\ossec.conf`), or a directory holding it:

```
luna-migrate wazuh /var/ossec/etc/ossec.conf \
  -out /etc/logrok-universal-agent/agent.yaml \
  -syslog-endpoint logs.example.net:6514 \
  -report wazuh-conversion.json
```

- The destination is always syslog, because the Wazuh manager link is not something this agent can speak.
  `-target native` (the default) and `-target syslog` therefore give the same result; give the receiver
  with `-syslog-endpoint host:port`. Without it the file carries `REPLACE_WITH_SYSLOG_HOST:6514` and the
  report lists it under "fill in before deploying". Output uses TLS; if your receiver's certificate comes
  from a private CA, add `ca_file:` to the `syslog` output. `-syslog-plain` sends plain TCP instead of TLS
  (a receiver without TLS, or a lab).
- `-windows` or `-linux` (at most one) converts for a Windows or a Unix host when the state paths and
  platform-dependent inputs cannot be told from the files.
- Wazuh allows more than one `<ossec_config>` block in one file. They are read in order.
- `-start-at end` (default) starts file inputs at new data so nothing is sent twice; `beginning` reads
  existing content.
- The output goes to `-out` (an existing file is never overwritten); the report summary always prints
  and `-report` also writes it as JSON. luna-migrate checks the configuration's structure, then runs the
  installed agent's `-check-config` (found beside `luna-migrate`, on the `PATH`, or named with `-agent`);
  a configuration the agent rejects is removed. Without an agent to hand it says so — run
  `logrok-universal-agent -check-config -config agent.yaml` on the target host.
- Environment references (`${VAR}`, `%VAR%` in a location) are never expanded: the input is reported and
  not converted. Write the resolved path and convert again.

### The shared agent.conf

A Wazuh manager can push a shared `agent.conf` (blocks `<agent_config name="…" os="…" profile="…">`) to
agents in a group. Give `luna-migrate` a directory that contains `ossec.conf` and the `agent.conf` next to
it (or under `shared/`) and it reads both. For each block:

- no selector — it applies to every agent of the group, so its `<localfile>` blocks are converted;
- `profile` — decided from the `<config-profile>` in `ossec.conf`;
- `name` — decided from the `<agent_name>` in the `<enrollment>` section of `ossec.conf`, when it is set;
- `os` — **cannot** be decided from the files.

A block whose selector cannot be decided is **listed, not applied**: the converter does not guess which
host it is running for. Tell it with the options below, or move that block's `<localfile>` blocks into
`ossec.conf` and convert again.

| Flag | Meaning |
|---|---|
| `-agent-conf FILE` | an extra shared `agent.conf` copied off the manager (repeat the flag, or comma-separate several) |
| `-os NAME` | this host's operating system, matched against `os=` (for example `Linux` or `Windows`); never guessed |
| `-profile NAME` | this agent's profile, matched against `profile=` (default: the `<config-profile>` in `ossec.conf`) |
| `-agent-name NAME` | this agent's name, matched against `name=` (default: the `<agent_name>` in `ossec.conf`) |

These four apply to the Wazuh converter only; the tool refuses them for any other source. Another
group's `agent.conf` found under `shared/` is listed the same way. `command` and `full_command` sources
delivered through `agent.conf` are ignored by Wazuh itself unless remote commands are enabled on the agent;
here they are reported like any other command source.

### Filters: `<ignore>` and `<restrict>`

Wazuh applies them to the log line (`ignore` wins over `restrict`). They become `expr` drop rules that
apply to their own input only — the agent's processors are global, so the converter stamps each such input
with a marker field `wazuh_input` and removes it again after the rules ran; it never reaches the receiver.

A pattern is converted only when it translates exactly:

- `PCRE2` (the default) patterns that the agent's regex engine accepts as they are. Look-around,
  back-references and possessive quantifiers are not available — those patterns are reported and, for
  `ignore`, the lines are forwarded.
- `osregex` (`\w \d \s \t \p \W \D \S \.`, the `+`/`*` modifiers on those, `^ $ |`) and `osmatch` (literal
  text, `^ $ |`, a leading `!` to negate). Anything outside the documented syntax is reported.
- If **any** `restrict` pattern of an input cannot be converted, no restriction is applied to that input
  (forwarding too much is safer than silently dropping what Wazuh kept), and the report says so.

The Wazuh reference does not say whether a pattern may match anywhere in the line or must match the whole
line; the converter searches anywhere. Check the first conversions against your own logs.

## What the report tells you to do by hand

Everything not converted is listed with the original line number and file:

| Wazuh | Result |
|---|---|
| `command`, `full_command` | **manual** — schedule the command with the OS, write to a file, tail the file |
| `multi-line:N`, `multiline_regex match="all"`, journald filters beyond unit / identifier / priority | **manual** — the input is converted without that part |
| `syslog-pipe` | **manual** — point the writer at the `syslog_in` input instead |
| relative `location` | **manual** — Wazuh resolves it against its install directory; write an absolute path |
| `<syscheck>`, `<rootcheck>`, `<sca>`, `<active-response>`, `<wodle …>`, and every other module | **skipped** — not log collection |
| `<client>` manager link and `<enrollment>` | **warning / skipped** — see the first section; no secret is copied |
| `/var/ossec/logs/active-responses.log` | **skipped** — written only by active response, which is not migrated |
| any directive the converter does not know | **manual** |

Nothing is dropped silently.

## Cut-over procedure

1. **Check** the Wazuh file with `wazuh-logcollector -t`, then **convert** with `luna-migrate wazuh`
   and read the whole report. Decide what you do about each `manual` line.
2. **Decide about the manager.** Once the Wazuh agent is stopped, the Wazuh manager receives nothing from
   this host. If you need its alerts, keep the Wazuh agent running.
3. **Run side by side** for a day if you can: start this agent, keep the Wazuh agent, and compare volumes
   at the receiver. Both only read the same files.
4. **Mind the position gap.** The Wazuh agent's read positions do not transfer. With `start_at: end` the
   agent starts at each file's end on first start; start it *before* you stop the Wazuh agent and accept a
   short double-delivery overlap — duplicates are cheaper than a gap. Windows event channels start from
   new events.
5. **Cut over:** stop and remove the Wazuh agent service (or only its log collection, if you keep the rest).
6. **Verify:** `curl -s localhost:9090/metrics` (when `service.metrics_listen` is set) shows events going
   out, and the receiver's volume matches what you had.

## Licensing note

The free Core tier covers file tailing, journald, the `expr` and `add_fields` processors and RFC 5424
syslog over TCP/TLS. The Windows Event Log, `linux_audit` and `oslog` inputs, and the disk store-and-forward
spool that the generated configuration enables, need an Apex entitlement — included at no charge with a
licensed logrok deployment. See the licensing section of the [configuration reference](../CONFIGURATION.md)
for the current split.
