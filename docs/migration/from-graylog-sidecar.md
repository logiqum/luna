# Migrating from Graylog Sidecar

The Graylog Sidecar does not collect logs itself. It is a **configuration manager**: it fetches collector
configurations from the Graylog server, writes them into a directory on the host, and starts and supervises
the collectors that use them (Filebeat, Winlogbeat and Auditbeat ship with it; others such as NXLog can be
added).
The collecting is done by those collectors.

So moving off the Sidecar means replacing *the collectors and the thing that configures them*. The agent
collects (files, Windows Event Log, journald, syslog…) and forwards, in one process, from one configuration
file. This guide converts what the Sidecar has already written on a host into that file.

## Is it a fit?

**Verified against:** the current Graylog Sidecar documentation and its 3.2 reference (read 2026-10-16). The Filebeat file was checked by Filebeat 8.15.3 and delivery into Logstash 8.15.3 and syslog-ng was run end to end; no run against a Graylog server or the Sidecar itself, and none against NXLog.

| If your Sidecar host… | Then |
|---|---|
| runs **Filebeat** and/or **Winlogbeat** configurations | converted (the Filebeat and Winlogbeat tables apply: [Filebeat](from-filebeat.md), [Winlogbeat](from-winlogbeat.md)) |
| runs an **NXLog** configuration | converted ([NXLog](from-nxlog.md) tables apply) |
| runs **Auditbeat**, Packetbeat, or another collector | **not converted**: the file is listed in the report as skipped. On Linux, the `linux_audit` input is the agent's way to collect kernel audit records |
| sends to a Graylog **Beats** input (Filebeat/Winlogbeat `output.logstash`) | converted to the agent's `beats` output, which speaks the same Beats protocol Logstash accepts |
| sends **GELF** to a Graylog GELF input (the usual NXLog setup) | **the agent has no GELF output.** The destination needs changing: add a Syslog TCP input in Graylog and point the agent at it, or send everything to one syslog receiver with `-target syslog` |
| depends on Graylog-side objects (inputs, streams, pipelines, extractors, content packs) | those live on the Graylog server and stay there; nothing on the host holds them |

## How it fits together

```
before:   files / Event Log ──► Filebeat | Winlogbeat | NXLog ──► Graylog (Beats / GELF / syslog input)
                                  ▲ configured and restarted by the Sidecar, from the Graylog server

after:    files / Event Log ──► agent ──► the same Beats destination, or a syslog receiver
                                  ▲ configured by one file (or by the agent's central management)
```

What replaces the Sidecar is **the agent's own configuration**: a file on the host, or the agent's central
management when you use it (register, configuration pull, heartbeat). There is no equivalent of assigning
collector configurations to hosts by tag in the Graylog web interface; with the Sidecar gone, a configuration
change is a change to the agent's configuration.

## Step 1 — convert

Get `luna-migrate` first: a separate download published with every agent release (`luna-migrate-<os>-<arch>`,
saved as `luna-migrate` / `luna-migrate.exe`) — see [Getting luna-migrate](../USER-GUIDE.md#getting-luna-migrate).

Run it on the Sidecar host. Give it the **generated directory**, or the **`sidecar.yml`** (it follows
`collector_configuration_directory` from there):

```bash
luna-migrate graylog /etc/graylog/sidecar/sidecar.yml \
  -out /etc/logrok-universal-agent/agent.yaml \
  -report graylog-conversion.json
# or name the directory directly:
luna-migrate graylog /var/lib/graylog-sidecar/generated -out agent.yaml
```

Graylog's reference lists `/var/lib/graylog-sidecar/generated` as the default directory on Linux and
`/etc/graylog/sidecar/sidecar.yml` as the default location of the settings file. Graylog's reference lists no
default for the generated directory on Windows; installations commonly keep it under the Sidecar's install
folder (for example `C:\Program Files\Graylog\sidecar\generated`) — check your `sidecar.yml`, or look where
the files are.

```powershell
& ".\luna-migrate.exe" graylog "C:\Program Files\Graylog\sidecar\sidecar.yml" `
  -out "C:\ProgramData\logrok-universal-agent\agent.yaml" `
  -report graylog-conversion.json
```

| Flag | Default | Meaning |
|---|---|---|
| `-out` | *(required)* | where to write the agent configuration; an existing file is never overwritten |
| `-report` | — | also write the full report as JSON |
| `-target` | `native` | `native`: each collector's own destination (a Beats destination becomes a `beats` output). `syslog`: everything to one RFC 5424 receiver over TLS |
| `-syslog-endpoint` | — | `host:port` of the syslog receiver for `-target syslog` |
| `-syslog-plain` | off | send the syslog output over plain TCP instead of TLS (a receiver without TLS, or a lab) |
| `-start-at` | `end` | where file inputs start on first run: `end` (only new lines: nothing already shipped is shipped again) or `beginning` |

luna-migrate then checks the structure and runs the installed agent's `-check-config` (found beside
`luna-migrate` or on the `PATH`, or named with `-agent`), which builds every module exactly as that agent
version would; a configuration the agent rejects is removed. Without an agent to hand it says so — then run
`logrok-universal-agent -check-config -config agent.yaml` on the target host.

The tool recognises each file in the directory **by its content**, not its name: an NXLog configuration by its
`<Input>` / `<Output>` blocks, a Filebeat or Winlogbeat configuration by its settings. It then converts each
with the rules in the Filebeat, Winlogbeat and NXLog guides and merges the results into **one** agent
configuration.

Things to know about the merge:

- **Module names carry the file's name** as a prefix (`filebeat_…`, `nxlog_…`), so nothing collides, including
  the files the agent keeps its read positions in.
- **Identical destinations are shared.** Two configurations sending to the same Beats address become one
  output.
- **Each file's events stay with that file's destinations.** When the files send to different places, the
  agent marks every event with a field `graylog_collector` (the file's name) and each output only takes its
  own file's events. That needs routing between several outputs, which is an **Apex** capability. The field is
  delivered with the event: strip it at the receiver if you do not want it.
- **A filter in one file does not touch another file's events.** Line filters and drop rules stay scoped to the
  collector they came from. A processor that cannot be limited that way (for example a JSON-parsing step that
  applied to all of one collector's events) is **left out and reported** rather than applied to everything.
- **A file that cannot be converted does not stop the others.** It appears in the report with the reason.
- If two collectors read the same Windows channel, each gets its own bookmark.

## Step 2 — read the report

| Severity | Meaning |
|---|---|
| `manual` | **not carried over**: a person must decide (each line says what to do instead) |
| `warning` | carried over, with a difference to check |
| `info` | carried over differently, nothing to do |
| `skipped` | deliberately not converted: a file that is not a Filebeat, Winlogbeat or NXLog configuration (Auditbeat, an XML file, an unrendered template, an empty file…) |

Every line of the report names the generated file it came from. Passwords and keys inside the collector
configurations are never copied: the generated file says `REPLACE_WITH_…` (search for it), and the report lists
them under "fill in before deploying". The **API token** in `sidecar.yml` is never read into the output or the
report; the agent does not need it, because it does not talk to the Graylog server.

Watch for these:

- **Variables that were not substituted.** A Sidecar template can contain `${sidecar.nodeName}`,
  `${sidecar.nodeId}` or your own `${user.…}` variables. The agent never expands `${…}`: if one is still in a
  generated file, it would be sent as that literal text. The report warns and names the line; replace it with
  the real value. A YAML file that still contains template text such as `{{ if … }}` is not valid configuration and
  is skipped. An NXLog file is recognised by its `<Input>` / `<Output>` blocks and is converted even if it still
  holds template text, so check the report for it.
- **GELF outputs (NXLog).** The agent has no GELF output. The report marks the output as `manual`: change the
  destination as described under "Is it a fit?".
- **The Sidecar's own fields.** Graylog's default Filebeat template adds `collector_node_id` and
  `gl2_source_collector` to every event. They are carried as ordinary fields on the events the agent sends. We
  have not run the agent against a Graylog server, so check on a test host that Graylog shows them as it does
  today.
- **Stale files.** The generated directory holds what was assigned to the host at the Sidecar's last
  synchronisation. If the host was reassigned in Graylog and the Sidecar was not running, convert after the
  Sidecar has synchronised, or check the files against the Graylog web interface.

## What is converted, and what is not

| Graylog Sidecar | Agent |
|---|---|
| Filebeat configuration in the generated directory | as in [Filebeat](from-filebeat.md): `filetail`, `journald`, `syslog_in`, … inputs; `beats` / `elasticsearch` / `kafka` output |
| Winlogbeat configuration | as in [Winlogbeat](from-winlogbeat.md): `windows_eventlog` inputs |
| NXLog configuration | as in [NXLog](from-nxlog.md): inputs, `syslog` output, routes |
| `sidecar.yml` `collector_configuration_directory` | followed to find the generated files |
| `sidecar.yml` `server_api_token` | never copied; not needed |
| `sidecar.yml` `server_url`, `tags`, `node_id`, `node_name`, `update_interval` … | not needed: reported (`server_url`, `tags`) or ignored |
| Auditbeat, Packetbeat and other collector configurations | **not converted** (reported as skipped) |
| The Sidecar's fetching, restarting and status reporting to Graylog | replaced by the agent's configuration (or its central management); no equivalent of the Graylog collectors page |
| Graylog inputs, streams, pipelines, extractors, content packs | server-side; unchanged by this migration |

## Step 3 — cut over without a gap

The Sidecar rewrites the collector configurations and restarts the collectors it supervises. **Stop and
disable the Sidecar service when the agent takes over**, or it keeps starting the old collectors.

1. Install the agent with the converted configuration and start it. With `start_at: end` it reads only lines
   written from now on.
2. Confirm at the destination that events arrive from the agent, with your fields.
3. Stop and disable the Sidecar service and the collectors it runs.

Between steps 1 and 3 both send new lines, so those few seconds arrive twice. Stopping first would lose the
lines written before the agent starts. The agent does not read the collectors' own position files.

## Sending to logrok instead

```bash
luna-migrate graylog /var/lib/graylog-sidecar/generated -out agent.yaml \
  -target syslog -syslog-endpoint logrok.example.com:16575
```

Inputs, fields and processors convert as above; only the destination changes (one shared syslog output, so no
routing field is needed). Add the TLS trust (`ca_file`, and a client certificate if your receiver requires one)
to the generated `syslog` output.

## Licensing

The converter is free. What the converted configuration uses follows the normal rules
([LICENSING](../LICENSING.md)): file tailing, journald, syslog input and syslog forwarding are Core; the
`beats`, `elasticsearch` and `kafka` outputs, the Windows Event Log input, routing between several outputs and
the disk spool are Apex.
