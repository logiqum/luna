# LUnA — Configuration reference

The agent reads one YAML file (`-config <path>`, or `LOGROK_UNIVERSAL_AGENT_CONFIG`, default `agent.yaml`).
A working template is [`configs/agent.example.yaml`](../configs/agent.example.yaml). The same schema applies on
every platform.

**Shape.** Top-level sections are `service`, `management`, `buffer`, `inputs`, `processors`, `outputs`. Each
`inputs` / `processors` / `outputs` entry has a `type`, an optional `name`, and then **that module's own
options inline** — adding a module type never changes the schema.

**Unknown option keys are surfaced, not silent.** A module option key it doesn't recognise (a typo, or a
removed/renamed option) is logged at **WARN** at startup — naming the module and the offending key — and the
option falls back to its default. Watch the startup logs for `ignoring unknown option key(s)`: a typo in a
reliability-critical key (`dead_letter_dir`, `on_reject`, `ack_timeout`, `use_ack`, …) otherwise silently runs
the default. The config still loads (an unknown key is not fatal), so a stale key on upgrade won't stop the
agent — but the warning tells you to fix it.

**Status key:** ✅ implemented · ◐ partial · 🔜 planned. Pipeline: `inputs → processors → outputs → buffer (on
write-fail)`. Multiple outputs fan out — every destination gets every event, each with its own spool (see
[`outputs`](#outputs)).

---

## `service`

| Key | Type | Default | Status | Notes |
|---|---|---|---|---|
| `mode` | string | `service` | ✅ | `service` (Windows Service / daemon) or `standalone` (foreground) |
| `log_level` | string | `info` | ✅ | `debug` \| `info` \| `warn` \| `error` (structured `slog` JSON) |
| `metrics_listen` | string | `""` (off) | ✅ | e.g. `127.0.0.1:9090` → serves Prometheus text at `/metrics` **and the live-sampling endpoint `/tap`** (see [Watching live events](USER-GUIDE.md#watching-live-events--tap)). **Bind localhost** unless scraped remotely — `/tap` is refused outright on a non-loopback bind, because it exposes event content. |

**Metrics exposed:** `logrok_agent_events_in_total`, `events_out_total`, `events_dropped_total`,
`backpressure_waits_total`, `spool_meta_write_errors_total`, `spool_write_errors_total`,
`udp_truncated_total`, `spool_tamper_detected_total` (counters),
`buffer_depth`, `last_forward_timestamp_seconds`, `spool_chain_legacy_segments`,
`license_state{state}` (0/1 per posture: core/licensed/grace/degraded), `license_grace_days_left`,
`license_info{tier,enforced}` (gauges). A non-zero
`spool_meta_write_errors_total` means the disk spool could not persist its read-head position (e.g. a
full or read-only spool disk) — its resume point may be stale after a restart. A non-zero-and-climbing
`spool_write_errors_total` means the spool disk is failing a batch **write/fsync** (not a capacity drop):
the pipeline retries that batch and never drops or commits it, so a transient fault (an ENOSPC that clears
as the drain frees space) self-heals, while a persistent fault (read-only or dead disk) backpressures the
inputs rather than losing data — even under `when_full: drop_newest`/`drop_oldest`, which govern the
configured **capacity** cap, not a hardware fault. `spool_tamper_detected_total`
counts tamper-evidence chain-verification failures **per observation, not per unique record** — the same
tampered record recounts every time it is re-read (a retried drain window, or a restart before the
segment holding it drains), so treat it as an "is tampering being observed" signal rather than a
distinct-record tally. `spool_chain_legacy_segments` is a **one-time census taken at agent start**
(recovery) of pre-chain spool segments (left over from before this feature shipped, or written by a run
where the chain key failed to load — a keyless run rotates into fresh legacy segments rather than
corrupting the chain, so a transient key outage is safe) — it does not decrement live as those segments
drain within a run; it ages toward zero across restarts as they're consumed. See `buffer.chain_key_file`
below.

## `management`

Central control plane. **Empty `endpoint` = unmanaged** (runs purely from the local file — valid for air-gapped
installs). Auth uses the logrok control-plane headers: `X-Api-Key` + `X-Tenant-Slug`. Per-agent **mTLS** hardens the channel on top of the api-key: set `tls.cert_file`/`tls.key_file` to present an operator-provisioned client certificate (`tls.mode: static`), or use `tls.mode: enrolled` to have the control plane issue and auto-renew a per-agent client certificate.

| Key | Type | Default | Status | Notes |
|---|---|---|---|---|
| `endpoint` | string | `""` | ✅ | `""` = unmanaged. Use HTTPS in production — the api-key is a bearer credential |
| `enrollment_token` | string | `""` | ✅ | **one-time** secret; exchanged at first start for `agent_id` + `api_key` (retried with backoff — collection runs regardless). Credentials persist in `state_path`, so the token can be removed after enrollment |
| `endpoint` value `auto` | string | — | ✅ | **discovery**: browse the local segment (mDNS/DNS-SD `_logrok-cp._tcp.local`) for a control plane instead of hard-coding a URL. Opt-in only — an agent with an endpoint written down is never re-pointed by the network. Discovery **locates, it never authenticates**: the enrollment token and TLS still prove the control plane, so a rogue advertiser only gets ignored. Finding nothing is not an error: the agent runs unmanaged for that run and says so. |
| `enrollment_bundle` | string | `""` | ✅ | path to a signed **air-gap enrollment bundle** (identity + entitlement + optional starting config), verified offline against the embedded issuer key. Adopted once (anchored by the bundle's own id) and then inert — safe to leave the media mounted. See [Enrolling an agent with no path to the control plane](USER-GUIDE.md#enrolling-an-agent-with-no-path-to-the-control-plane-bundles). |
| `state_path` | string | `agent-state.json` next to the config | ✅ | where enrollment credentials + applied config version persist (atomic, `0600`). Secrets live here, never in the config file |
| `api_key` | string | `""` | ✅ | manual provisioning alternative to enrollment; sent as `X-Api-Key` (enrolled state takes precedence) |
| `tenant_slug` | string | `""` | ✅ | sent as `X-Tenant-Slug` |
| `heartbeat_every` | duration | `30s` | ✅ | heartbeat: agent_id, hostname, platform, version, config version, events/min, buffer depth |
| `config_poll_every` | duration | `60s` | ✅ | config-pull cadence (ETag/If-None-Match — unchanged config costs a 304) |
| `tls.mode` | string | `static` | ✅ | `static` (operator-provided cert files) or `enrolled` (the control plane issues and the agent **auto-renews** a per-agent client certificate — requires `endpoint` + enrollment, and a control plane with the agent-cert CSR endpoint). The key/cert/chain persist next to `state_path` as `agent-cert.pem`/`agent-key.pem`/`agent-ca.pem` (`0600`); the private key never leaves the host. Applies to both the **control-plane** channel and **data-plane** outputs that opt in with `cert_source: enrolled` |
| `tls.ca_file` | string | `""` | ✅ | PEM CA bundle to verify the control-plane server (pins the server cert). Empty = system roots |
| `tls.cert_file` | string | `""` | ✅ | client certificate (PEM) presented to the control plane for **mTLS** (set with `key_file`) |
| `tls.key_file` | string | `""` | ✅ | client private key (PEM) for mTLS |
| `tls.server_name` | string | `""` | ✅ | SNI / verification name override |
| `tls.insecure_skip_verify` | bool | `false` | ✅ | **dev only** — disables server verification |

**Remote config + hot-reload:** when the control plane serves a new config version, the agent validates it
(same parser as local load), **refuses** configs that fail validation or would drop `management.endpoint`
(lock-out guardrail), writes it atomically over the config file, then reloads: the pipeline stops gracefully
(in-flight events spool — at-least-once), and the agent rebuilds from the new file. No process restart needed.

> Central management (enroll, config-pull, heartbeat) works against the logrok control plane or any
> compatible implementation of this contract; leave `management` unset to run fully unmanaged.

## `buffer`

Disk-backed store-and-forward — the air-gap guarantee.

| Key | Type | Default | Status | Notes |
|---|---|---|---|---|
| `enabled` | bool | `false` | ✅ | turn the spool on |
| `dir` | string | `""` | ✅ | spool directory. **Set this** to survive restarts; empty → in-memory only (lost on restart). |
| `max_bytes` | int | `0` | ✅ | on-disk cap; `0` = unbounded. Behaviour at the cap is set by `when_full`. Honored for any positive value; a very small cap (below ~64 KiB) may transiently hold up to one ~64 KiB segment. |
| `when_full` | string | `drop_oldest` | ✅ | `drop_oldest` (discard oldest spooled, keep freshest) \| `drop_newest` (refuse new, keep oldest) \| `block` (back-pressure the source until space frees — no loss). |
| `flush_every` | duration | `2s` | ◐ | drain cadence |
| `chain_key_file` | string | `""` (auto-resolved) | ✅ | overrides where the tamper-evidence chain key lives. Empty = resolved automatically (see below). |

On disk: fsync'd, **CRC32-framed** NDJSON segment files (`<crc> <json>` per line, or `<crc> <mac> <json>` on a
chained segment — see below) + a `head.json` read cursor (atomic rename). A torn or bit-rotted record fails its
CRC and is skipped on replay (the drain doesn't stall); a torn tail from a crash mid-append is truncated on
recover. When `when_full: block`, the agent throttles its inputs once the spool is full rather than dropping —
watch `backpressure_waits_total` to see it engage. Under `when_full: drop_oldest`, events discarded to honor
`max_bytes` are counted in the dropped-events metric and logged, so cap-overflow loss is visible rather than
silent.

**If store-and-forward is withdrawn (licensing).** The disk spool is an Apex capability. When an agent is
running without a covering entitlement *and* enforcement is active, the spool switches to **drain-only**
rather than being switched off: it is still opened, and every event already written to it is delivered
normally, but it accepts no new events. You will see one warning at start and a single
`spool drain complete` line once the existing backlog has been delivered. While the agent is drain-only,
Core forwarding is unaffected — events go straight to the output as usual; they are only lost if the output
is unavailable, and each such loss is counted in `logrok_agent_events_dropped_total` and logged, never
silent. The spool directory and its contents are never deleted. This behavior applies whenever licensing
enforcement withdraws the spool capability (see [LICENSING.md](LICENSING.md)).

### Tamper-evident chaining (on by default)

Whenever the disk spool (`dir`) is in use, every record spooled to disk is bound into a **keyed HMAC-SHA-256
chain**: `mac_i = HMAC(key, mac_{i-1} || json-bytes)`, and each segment opens with a self-authenticating
`#chain` header binding it to the previous segment's final MAC. This is **on automatically** — there is no
config toggle to turn it off (short of the key becoming unavailable, below) — so the claim "every deployed
agent's spool is tamper-evident" holds fleet-wide without per-agent setup.

- **Key.** 32 random bytes, hex-encoded, file mode `0600`, auto-generated on first use if absent. Resolution
  order (shared byte-for-byte between the running agent and the `-verify-spool` CLI so the two can never disagree about where the key is):
  1. `buffer.chain_key_file`, if set;
  2. next to `management.state_path` as `spool-chain.key`, if management is configured;
  3. a sibling of the spool `dir`, named `<dir>.chain.key` (e.g. `dir: /var/lib/agent/spool` →
     `/var/lib/agent/spool.chain.key`).

  The key is **deliberately never stored inside the spool directory** — an offline copy or backup of the
  spool must not carry the key that authenticates it. A key file that exists but doesn't parse as a 32-byte
  hex key is a **hard error at spool open** (never silently replaced — that would orphan the existing chain).
  If the resolved key path can't be *created* (e.g. a read-only state directory), the spool still opens —
  forwarding must never go down because a key couldn't be written — but chaining is **disabled for that run**
  with a loud warning log; watch for it. A keyless run is **chain-safe**: it rotates into fresh **legacy**
  segments instead of appending unchained records into a chained one, so a transient key outage never
  corrupts the chain or produces false tamper flags — the outage's segments are censused
  (`spool_chain_legacy_segments`), drain unverified, and age out.
- **Detection, not prevention.** The chain is a keyed MAC, not a cryptographic seal against every attacker: an
  attacker who can **read the key file** can re-forge a self-consistent chain. The guarantee targets an
  attacker who can modify spool files (or an offline copy/backup of them) but cannot read the key. See the
  full honest-limitations list in [USER-GUIDE.md § Auditing the spool](USER-GUIDE.md#auditing-the-spool).
- **Runtime detection** (a record or segment fails chain verification during drain): the event is still
  **delivered** — never withheld from the SIEM — with `spool_tamper="true"` added to its fields, plus an
  `slog.Error` log line, the `spool_tamper_detected_total` metric (above), and a **sticky** heartbeat
  `metadata.spool_tamper: true` that stays `true` until the agent restarts or a config hot-reload rebuilds the
  management client (it does not clear itself just because a later drain window comes back clean).
- **Offline audit — `-verify-spool`.** `logrok-universal-agent -verify-spool <dir> [-chain-key <file>]` walks
  every segment oldest→newest, checking header authenticity, cross-segment linkage, every record's MAC, and
  (when `head.json` carries an authenticated tail anchor) that the anchor is reachable in the chain.
  `-chain-key` defaults to the same key the running agent would resolve (derived from `-config`), so an
  auditor doesn't need to know the key path by hand. It prints one verdict word and exits accordingly:

  | Exit | Verdict | Meaning |
  |---|---|---|
  | `0` | `INTACT` | every chained segment verified; no tamper evidence found |
  | `1` | `TAMPER EVIDENCE` | at least one verification failure — the report names the first divergence (segment, line, what failed) and a count |
  | `2` | `UNVERIFIABLE` | the spool has no chained segments — all-legacy or an empty/nonexistent-content directory (0 segments is reported as "fully drained or wiped" because those two are indistinguishable) — or the chain key file is missing (the CLI never creates keys; copy the agent's key for audits) |

  An **operational** failure (bad `-verify-spool` path, unreadable `-config`, a garbage key file) is a
  separate, loud stderr message and also exits `1` — it is **not** a tamper verdict sharing that code by
  coincidence of severity, not meaning; read the printed line, don't infer tampering from the exit code
  alone. Run it against a **stopped** agent, or a copy of the spool directory, for an authoritative result —
  against a live agent it's best-effort (a concurrent append can show up as a benign torn tail) and the
  output says so.

---

## `licensing`

Points the agent at its license. **Optional — omit it and the agent runs the free
Core tier at any scale** (also the case for the managed "free with logrok" path, where the
entitlement is delivered over enrollment). A config using Apex-tier capabilities without a
covering entitlement has those modules disabled at load (degrade-to-Core, after any grace
window) — Core capabilities are never blocked and the agent never exits over licensing.
Model, tiers, and license states: [LICENSING.md](LICENSING.md).

| Key | Type | Default | Status | Notes |
|---|---|---|---|---|
| `license_file` | string | `""` | ✅ | path to a signed offline license (`.lic`). Verified **offline** (Ed25519, no phone-home). |

The agent computes a license **posture** at every config load — `core` (no gated feature in use),
`licensed`, `grace` (expired but within the license's grace window, default 30 days), or `degraded`
(unlicensed gated features) — and reports it live on `/metrics` and, for managed agents, on every
heartbeat. An unreadable/invalid license falls back to the free Core entitlement. A valid license is
required for licensed features; free Core features are always available in every posture.

### Modules newly marked Apex — notice period, nothing stops working yet

The modules below are Apex, and are marked **Apex** in their reference sections above. Earlier
releases did not mark them and did not check them, so a configuration built on this release's
predecessors may be using them without an entitlement:

| | Modules |
|---|---|
| Inputs | `etw` · `wmi` · `linux_audit` · `mqtt_in` |
| Processors | `redact` · `lookup` · `adaptive_sample` |
| Outputs | `snare` · `hec` · `sentinel` · `xsiam` · `kafka` · `s3` · `loki` · `otlp` |

**Nothing is disabled in this release.** If your entitlement does not cover them, the agent logs a
warning naming the modules, reports posture `degraded`, and **keeps running every one of them**. They
are disabled only in the next **major** version — so you have a full release line to add a license
(`license_file`), enroll with a logrok control plane, or change the configuration.

The free **Core** tier is unchanged and is exactly what it has always been: file tail, syslog-in,
journald, HTTP-in, the parsers (`parse_json`/`parse_csv`/`parse_kv`/`parse_xml`/`expr`/`add_fields`/
`filter`), RFC 5424 syslog output over TCP/TLS, and `/metrics`. Nothing was taken out of Core. Core
forwarding is never blocked in any posture, and TLS is never gated.

To check a configuration before upgrading, start the agent and read the startup log: a `licensing:`
warning naming modules is the notice; no such line means the configuration is unaffected.

---

## `inputs`

### `windows_eventlog` ✅ *(verified end-to-end on real Windows hardware)* — Windows only

Collects the **Windows Event Log** — the OS's structured, durable record of system, security, and
application activity (logons, service installs, process creation, application errors). Point `channels`
at any channel — the classic three, or modern `Microsoft-Windows-*/Operational` channels like Sysmon —
and each event arrives with its structured data as fields plus either the publisher's rendered
human-readable message or the raw event XML (`render`). Live subscription with persistent resume
bookmarks is the normal mode; `file:` instead drains saved `.evtx`/`.evt` files — the forensic /
offline-analysis path. This is the default input for any Windows endpoint.

| Option | Type | Default | Notes |
|---|---|---|---|
| `channels` | []string | `[Application, System, Security]` | channels to subscribe to (live). Mutually exclusive with `file` |
| `file` | string | `""` | offline mode: path or glob of saved event-log file(s) (`.evtx` / legacy `.evt`) to drain instead of subscribing live — the forensic / air-gapped / offline-analysis path (NXLog's `File` directive). Opened with `EvtQuery`; the same `query`, `render`, and field extraction apply. A **one-shot drain** (reads every event, then that reader is done — not a live follow) and reproducible, so it persists **no** resume bookmark. Mutually exclusive with `channels` (setting both is a config error) |
| `bookmark_path` | string | `C:\ProgramData\logrok-universal-agent\bookmarks` | per-channel resume bookmarks (live mode only; unused in `file` mode) |
| `query` | string | `""` | optional XPath / structured query filter (applies to both live and `file` mode) |
| `render` | string | `message` | payload form for the event body: `message` (the publisher's rendered human-readable text) or `xml` (the raw native Windows Event XML — the `<Event xmlns="…/events/event">…</Event>` document `EvtRender`/`wevtutil /f:xml` produce, matching what the syslog-ng Windows Agent emits). Structured fields are attached to the event in **both** modes; `render` only changes the body. Pairs naturally with the `otlp` output over gRPC to forward Windows events as XML to a Linux collector. |

Resumes after the saved bookmark, else starts from future events on first run. The Security channel needs
admin / `SeSecurityPrivilege`; without it that channel is skipped with a warning (others continue). On non-Windows
builds this is a stub that errors if started.

Each channel's bookmark filename is now derived injectively (a short hash of the raw channel name is appended
when the name contains `/`, `\`, `:` or spaces), so channels that differ only by separator no longer share one
bookmark file. Simple channel names (`Security`, `Application`, `System`) keep their legacy filename and resume
cleanly. Channels **with** separators (e.g. `Microsoft-Windows-Sysmon/Operational`) get a new filename, so on
first start after upgrade they resume from scratch once — a bounded, at-least-once-safe duplicate replay.

Emits `event_id`/`provider`/`channel`/`record_id`/`pid`/`thread_id` from the trusted `<System>` block, plus
each `EventData` entry as a field (named entries keep their name; unnamed legacy ones are keyed `param1`,
`param2`, …). `pid` is the *emitting* process (the `<Execution>` ProcessID, not any subject process an
`EventData` field may name) and also rides in RFC 5424 PROCID — see [`syslog` output](#syslog---rfc-5424-over-tcp-tlsmtls-or-udp-diode-mode).

**Reserved-field protection.** Those six `<System>`-derived names are reserved. If an `EventData`/`UserData`
entry uses one of them as its data name (case-insensitively — e.g. `<Data Name="event_id">` or
`Name="Channel"`), it does **not** overwrite the trusted value; instead it is preserved under an
`eventdata_`-prefixed key (`eventdata_event_id`, `eventdata_Channel`, …). This keeps a record from forging the
exact fields detections key on (spoofing `event_id=4624` while the real EventID differed, or faking
Channel/Provider). The original casing of the data name is preserved after the prefix.

**UserData events.** Providers such as `TerminalServices-LocalSessionManager` (RDP logon EventIDs 21/24/25 with
`User`/`SessionID`/`Address`), AppLocker and DHCP emit their structured data under `<UserData><EventXML>…`
instead of `<EventData>`. When an event has no `EventData`, the agent generically flattens the `UserData` leaf
elements into fields (element name → value, same reserved-field protection applies) — no per-provider schema.

The **message** is the publisher's fully rendered human-readable text (`EvtFormatMessage`, the same text the
Event Viewer shows); when a provider's message template isn't available (unregistered provider / no message
DLL) the agent falls back to a compact `name=value` summary of the event's `EventData` (or `UserData` fields
when there is no `EventData`), so an event is never forwarded with an empty message.

**Sysmon:** `Microsoft-Windows-Sysmon/Operational` is a regular Event Log channel — point `channels` at it
to collect Sysmon security telemetry (process creation, network connections, DNS, registry). Annotated
preset with event-ID reference and noise-reduction examples: [`configs/sysmon.example.yaml`](../configs/sysmon.example.yaml);
walkthrough in the [User Guide](USER-GUIDE.md#windows-security-telemetry--sysmon).

**Windows Event Forwarding (WEF):** if you already run a WEF collector (WEC), its `ForwardedEvents`
channel is collectible the same way — `channels: [ForwardedEvents]` on the WEC host drains the whole
WEF estate through one agent. (Acting as a WEC *server* is out of scope; per-endpoint agents replace WEF.)

### `etw` ✅ *(runtime-verified on real Windows hardware)* — Windows only — **Apex**

Consumes **ETW (Event Tracing for Windows)** providers through an agent-owned **real-time trace session** —
the high-volume kernel/analytic telemetry that never reaches the Event Log (kernel process/network activity,
DNS Client analytic tracing, Time/Update service traces; the NXLog `im_etw` engine gap). Hand-rolled
`StartTrace`/`EnableTraceEx2`/`OpenTrace`/`ProcessTrace` + TDH syscall layer over `golang.org/x/sys/windows`
— still no cgo, still one static binary. On non-Windows builds this is a stub that validates config but
errors if started.

| Option | Type | Default | Notes |
|---|---|---|---|
| `providers` | []string | *(required)* | providers to enable — each a **GUID** (`{xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}`, braces optional) or a registered **manifest provider name** (e.g. `Microsoft-Windows-DNS-Client`), resolved at start via `TdhEnumerateProviders`. A name that doesn't resolve is a start error telling you to use the GUID (`logman query providers` lists both) |
| `session_name` | string | `logrok-agent-etw` | the trace-session name the agent owns. A stale same-named session left by a previous crash is stopped and reclaimed automatically at start; the session is always stopped on shutdown |
| `level` | int \| string | `4` | max event verbosity: `1`–`5` or `critical`/`error`/`warning`/`info`/`verbose` |
| `match_any_keyword` | int \| string | `0` (all) | provider keyword mask (`EnableTraceEx2` MatchAnyKeyword); accepts an integer or a hex string like `"0x10"` — use the hex-string form for high-bit masks that overflow YAML integers |
| `buffer_events` | int | `4096` | size of the internal hand-off buffer between the ETW callback and the pipeline (see the drop note below) |

**Privileges (required).** Real-time ETW sessions need **Administrator** or membership in the
**Performance Log Users** group. Without either, the input fails at start with an explicit
"requires Administrator or Performance Log Users membership" error — there is no degraded mode.

**Backpressure = count-and-drop (documented behavior).** The ETW callback runs on the trace's own thread and
must never block — stalling it backs up the kernel's trace buffers and makes **ETW itself** drop events
silently and unaccounted. So when the pipeline can't keep up and the internal `buffer_events` buffer is full,
the input **counts and drops** instead (warning log with the running count, first drop + every 10,000th, and
a shutdown total). Size `buffer_events` up, lower `level`, or narrow `match_any_keyword` for chatty providers.
This is unlike the pull-based inputs (eventlog/filetail), which apply backpressure losslessly.

**Decoding: manifest providers, scalar properties (honest subset).** Events are decoded via TDH
(`TdhGetEventInformation`) — this covers **manifest-based providers** (the `Microsoft-Windows-*` estate).
Top-level **scalar** properties are decoded: Unicode/ANSI strings, signed/unsigned 8–64-bit integers,
float/double, boolean, GUID, pointer, FILETIME/SYSTEMTIME, SID, hex int32/64, and fixed-length binary
(hex-encoded). **Structs, arrays, and param-sized properties are not decoded** — decoding of an event stops
at the first such property (one debug log per provider) and the fields decoded up to that point are kept.
**WPP and TraceLogging-only providers are not decoded**: their events arrive with header fields only (a
GUID-labeled provider and the summary message). Kernel *logger* ("NT Kernel Logger" / MOF) tracing is not
supported — use the modern manifest equivalents (e.g. `Microsoft-Windows-Kernel-Process`).

The **message** is the provider's TDH event-message template with `%1…%N` property inserts substituted
(plus `%%`/`%n`/`%t` escapes); when a provider/event has no template, a compact
`provider/event-id: name=value …` summary of the decoded properties is used, so an event is never forwarded
with an empty message.

Emits `provider` (name, or GUID when unresolvable), `etw_event_id`, `etw_level`, `etw_keywords` (hex),
`etw_opcode`, `etw_task`, `pid`, `tid` from the trusted event header, plus each decoded property as a field.
A property whose name collides (case-insensitively) with one of those reserved header-derived keys is
preserved under an `etwdata_`-prefixed key instead of overwriting the trusted value — the same anti-spoofing
rule as `windows_eventlog`'s reserved-field protection. Severity maps from the ETW level
(critical→2, error→3, warning→4, info→6, verbose→7), facility is local0 (16), the timestamp comes from the
event header (FILETIME), and `host` is the agent's hostname.

```yaml
inputs:
  - type: etw
    providers:
      - "{22FB2CD6-0E7B-422B-A0C7-2FAD1FD0E716}"   # Microsoft-Windows-Kernel-Process
      - Microsoft-Windows-DNS-Client                 # names resolve via TDH
    level: info
    match_any_keyword: "0x10"                        # WINEVENT_KEYWORD_PROCESS
```

### `wmi` ✅ *(compile+unit-verified; runtime-validated on Windows hardware)* — Windows only — **Apex**

Polls **WMI (Windows Management Instrumentation)** with a **WQL** query and forwards each returned CIM
instance as an event — the NXLog `im_wmi` parity input. WMI is Windows' system-inventory and live-state
surface: running services, disks, hotfixes/patches, hardware, process lists, network config. WQL is its
SQL-ish query language (`SELECT … FROM <class> WHERE …`). Reach for `wmi` when the signal you need is
**state you have to ask for on an interval** — "which services are running", "what OS build is this",
"is this disk near full" — rather than a stream of records in an Event Log channel (`windows_eventlog`) or
a real-time trace (`etw`). Under the hood it shells out to **PowerShell**
(`Get-CimInstance -Query … | ConvertTo-Json`) once per poll and parses the JSON — the same cgo-free
subprocess pattern the macOS `oslog` input uses. The tradeoff: it depends on PowerShell being present (it is,
on every supported Windows) and pays a subprocess spawn per poll, in exchange for staying a single static
cgo-free binary with no COM syscall layer. A future native-COM reader is a possible enhancement. On non-Windows
builds this is a stub that validates config but errors if started.

| Option | Type | Default | Notes |
|---|---|---|---|
| `query` | string | *(required)* | the **WQL** query, e.g. `SELECT Caption,Version FROM Win32_OperatingSystem`. Single quotes inside it (e.g. `WHERE State='Running'`) are preserved — they're passed through as PowerShell literal-escaped text |
| `namespace` | string | `root/cimv2` | the WMI namespace to query (most classes live in `root/cimv2`; some newer ones in `root/StandardCimv2`, etc.) |
| `interval` | duration | `60s` | how often to run the query. WMI is a *poll*, not a stream — each interval re-runs the query and emits the current result set |
| `source` | string | `wmi` | the `Event.Source` label for emitted events |
| `facility` | int | `16` | syslog facility (a WMI instance has none); `16`=local0, matching eventlog/etw/journald |
| `severity` | int | `6` | syslog severity (`6`=info) |

Each returned CIM instance becomes one event; each of its properties becomes a field (flattened one level —
a scalar becomes its string form; a nested object/array is re-encoded as compact JSON, capped by
`ConvertTo-Json -Depth 3`). **Noise-key skip:** the PowerShell/CIM bookkeeping keys `PSComputerName`,
`PSShowComputerName`, `CimClass`, `CimInstanceProperties`, and `CimSystemProperties` that ConvertTo-Json adds
to every instance are dropped — they carry no telemetry. Field keys are kept verbatim; a would-be collision is
namespaced (`key_1`, `key_2`), never clobbered. **`Message`** is a compact, key-sorted `k=v` summary of the
instance's properties (WMI/CIM instances carry no natural message text), capped at 1024 bytes on a rune
boundary. **`Timestamp` is set to collection time (now)** — WMI instances have no uniform event time (most
classes expose no timestamp at all), so now is the honest, uniform choice. `Host` is the agent's
`os.Hostname()`. Source/facility/severity come from config.

An **empty result set is zero events, not an error**. A **WQL/PowerShell error** (bad query, missing
namespace, access denied) makes the first poll fail loudly at startup (a misconfiguration surfaces, doesn't
silently spin); a later transient poll failure is logged and retried on the next interval. Garbage output from
a poll that otherwise exited cleanly is skipped and counted (warning log), never crashing the input.

```yaml
inputs:
  - type: wmi
    name: running-services
    query: "SELECT Name,DisplayName,State,ProcessId FROM Win32_Service WHERE State='Running'"
    namespace: root/cimv2   # default
    interval: 60s           # WMI is polled, not streamed
```

### `filetail` ✅ — cross-platform

Tails plain-text log files, emitting one event per line — the universal input for anything that writes a
log file: application logs, IIS/web-server logs, appliance exports, Kubernetes container logs. It follows
globs, survives rotation (rename or in-place truncation) and agent restarts (offset checkpoints via
`state_path`), assembles multiline records such as stack traces, and decodes UTF-16 (the Windows norm for
PowerShell redirects and `wevtutil` exports). If a program logs to a file, this is the input to reach for.

| Option | Type | Default | Notes |
|---|---|---|---|
| `path` | string | `""` | a single file or glob to tail (`path` or `paths` required) |
| `paths` | []string | `[]` | one or more globs. Supports `*` `?` `[…]` (within a path segment) and recursive `**` (whole segment, e.g. `/var/log/**/*.log`) |
| `exclude` | []string | `[]` | globs to skip (same syntax) |
| `start_at` | string | `end` | `end` (new lines only) or `beginning` (whole file) — applies only when there is no saved checkpoint for the file |
| `state_path` | string | `""` | offset-checkpoint file for resume-after-restart. Empty = no persistence (restart re-applies `start_at`). |
| `poll_interval` | duration | `500ms` | how often to poll for new data |
| `fingerprint_size` | int | `1024` | bytes hashed to identify a file across rename-rotation. A file smaller than this has no rename-stable identity yet; its resume checkpoint is still persisted, guarded by a hash of everything already consumed — on restart the agent resumes exactly where it stopped if (and only if) the file's consumed bytes are unchanged, and otherwise re-applies `start_at` (duplication at worst, never loss). Once the file grows to this size its full fingerprint is adopted and the checkpoint follows it |
| `facility` | int | `16` | syslog facility for emitted events (a file has none); `16`=local0, matching eventlog/journald |
| `severity` | int | `6` | syslog severity for emitted events (`6`=info) |
| `encoding` | string | `utf-8` | `utf-8`, `utf-16le`, `utf-16be`, or `auto` (per-file BOM sniff — a mixed UTF-8/UTF-16 glob just works). UTF-16 is the Windows norm (PowerShell redirects, wevtutil exports); decoded to UTF-8 at the edge, BOM stripped, CRLF trimmed. Newline detection is code-unit-aware (a `0x0A` byte inside a character never splits a line); checkpoints stay raw byte offsets |
| `multiline.line_start_pattern` | string | `""` | regex; a matching line **begins** a new event, non-matching lines are continuations |
| `multiline.line_end_pattern` | string | `""` | regex; lines accumulate until one **matches**, which closes the event (set start *or* end, not both) |
| `multiline.flush_timeout` | duration | `5s` | emit a still-open multiline event after this much inactivity |

Handles in-place truncation and rename-rotation (finishes the old file, then reads the new one from the start).
An in-place rewrite of a file **smaller than `fingerprint_size`** (a same-size status-file rewrite, or a
`copytruncate`/truncate+rewrite that completes within one poll and never appears at size 0) is detected via the
file's head bytes and re-read from the start — such a rewrite has no stable fingerprint to compare, so it would
otherwise be missed. A plain append to a small file is not mistaken for a rewrite.
**Compressed rotated archives**: a glob match ending in `.gz` is read **once, in full** (each line one event,
UTF-8, no multiline assembly) and checkpointed as consumed — the logrotate `compress` case, where lines written
just before rotation only exist inside the archive. The consumed marker is written only after **every** event
of the archive has been durably handled by the pipeline (spooled/delivered), not merely read. A corrupt or
still-being-written `.gz` is skipped and retried; a crash before the archive fully committed re-reads that
archive (at-least-once; the dup window is one archive).
The checkpoint trails what's been forwarded, so a crash re-reads a bounded tail (possible dups) rather than
skipping lines — **resume with a bounded dup/loss window at the crash boundary, not exactly-once** (no output
ack — see the at-least-once note under [`buffer`](#buffer)). Multiple files / recursive glob / exclude are supported via
`paths`/`exclude`, and multiline assembly via `multiline.line_start_pattern`/`line_end_pattern` (the checkpoint
stays at the open record's start, so a crash re-reads the in-flight record rather than splitting it).

#### `format` (container logs)

`format: text | cri | docker | auto` (default `text`). When set to `cri`/`docker`/`auto`,
filetail parses Kubernetes container logs and reassembles partial (`P` → `F`) lines into one
event. `auto` detects per-file from the first line (`{` ⇒ docker JSON, else CRI text).
Mutually exclusive with `multiline`.

- `max_merged_bytes` (default `1048576`) — cap on a reassembled multi-line entry; on overflow
  the event is truncated and gets `cri_truncated="true"`.
- `add_k8s_metadata` (default `true` when `format` is set) — extract `k8s_namespace`, `k8s_pod`,
  `k8s_container`, `container_id` from the kubelet log path.
- `node_env` (default `KUBE_NODE_NAME`) — env var read into the `k8s_node` field.
- `flush_timeout` (duration, default `5s`) — idle window before an orphaned partial (a `P`-tagged
  line whose closing `F` never arrived — a truncated/rotated container write) is emitted so it is
  never held indefinitely. The CRI-mode equivalent of `multiline.flush_timeout` (the two modes are
  mutually exclusive, so this key is read at the top level here).

Emitted fields: `stream` (stdout/stderr), `k8s_*`/`container_id` (when metadata is on),
`cri_truncated`/`cri_partial` flags on truncated or orphaned-partial records. The raw log
content is the event message; everything else rides as RFC 5424 structured-data fields.

### `syslog_in` ✅ — cross-platform (the gateway ingress)

Listens for syslog and turns each received message into a pipeline event. Use it to make the agent a
**gateway**: network gear, appliances, and IoT/OT devices that can't run an agent point their syslog
output here, and the gateway adds the buffering, TLS, and central management those devices lack. Accepts
RFC 5424 and RFC 3164 over TCP and/or UDP, and (on unix platforms) a local unix-domain socket as a
`/dev/log` replacement.

| Option | Type | Default | Notes |
|---|---|---|---|
| `listen` | string | `""` | `host:port`, e.g. `0.0.0.0:5514` (`listen` and/or `socket` required) |
| `protocols` | []string | `[tcp, udp]` | any of `tcp`, `udp` (applies to `listen`) |
| `socket` | string | `""` | **unix platforms**: receive local syslog on a unix-domain socket (NXLog `im_uds` parity) — a `/dev/log` replacement on hosts with no syslog daemon. A stale socket file from a crash is replaced; a non-socket file at the path is **refused**, never deleted |
| `socket_protocol` | string | `unixgram` | `unixgram` (datagram — what `syslog(3)`/`logger` speak) or `unix` (stream) |
| `socket_mode` | string | `0666` | octal permission bits on the socket file (default lets unprivileged local writers log) |

Parses RFC 5424 (PRI→severity/facility, timestamp, host, `app`/`procid`/`msgid`; **structured-data exploded into
fields** — each `PARAM` becomes a field plus a joined `sd_id`) and RFC 3164 (PRI + **`Mmm dd HH:MM:SS` timestamp
and hostname extracted**, tag+body kept as the message; assumes the current year). TCP auto-detects RFC 6587
octet-counting vs newline framing. Unparseable input is kept verbatim as the message. This is how devices/MCUs
that can't run the agent feed a gateway.

**Listener hardening:** the TCP/stream-unix path bounds each frame (1 MiB) and enforces a **5-minute idle read
deadline** — a peer that stalls mid-message or opens a connection and sends nothing is closed, so an
unauthenticated slow client can't pin connections open (slow-loris). A legitimately low-volume sender that
delivers a complete message within the window is never dropped. The deadline is a fixed default, not
configurable.

### `relay_in` ✅ — cross-platform (the ack'd-transport receiver)

Listens for the agent's own acknowledged relay protocol — the receiving half of an agent→agent hop (the
sending half is the [`relay` output](#relay---ackd-reliable-transport-agentagent)). Run it on a
gateway/concentrator agent when a plain syslog/TCP hop isn't trustworthy enough: TCP can report a write
as sent while the peer loses the last events on a broken connection, whereas here every batch gets an
application-level delivery confirmation.

| Option | Type | Default | Notes |
|---|---|---|---|
| `listen` | string | *(required)* | `host:port`, e.g. `0.0.0.0:16576` |
| `tls_cert_file` | string | `""` | serve TLS (with `tls_key_file`); strongly recommended outside a lab |
| `tls_key_file` | string | `""` | server key (with `tls_cert_file`) |
| `tls_client_ca_file` | string | `""` | **mTLS**: require + verify client certificates against this CA (needs the server cert/key set) |

The receiving end of the [`relay` output](#relay---ackd-reliable-transport-agentagent) — the agent's ack'd
transport. Run it on a **gateway/concentrator agent**: edge agents point `relay` outputs at it, it ACKs each
batch only after every event in it has been **durably handled** by its own pipeline (spooled / delivered /
dropped), never merely accepted onto the pipeline channel — so an already-ACKed sender never discards events
the gateway then loses in a crash before spooling them. It dedups frames replayed after a lost ACK and applies
real end-to-end backpressure (a full pipeline stalls the ACK, which stalls senders into their own spools).
Loss-free by design where plain syslog/TCP is fire-and-forget. (On a RAM-only spool the durable point is
delivery — the documented at-least-once-with-disk boundary.)

### `http_in` ✅ — cross-platform (HTTP(S) ingress)

Accepts log lines over HTTP(S) `POST` — the push ingress for sources that speak HTTP but can't run the
agent or emit syslog: webhooks, IoT/edge devices, app log shippers. Each newline-delimited line of the
body becomes one event; bearer-token auth, TLS/mTLS, and a body-size cap guard the endpoint. The HTTP
analogue of `syslog_in`.

| Option | Type | Default | Notes |
|---|---|---|---|
| `listen` | string | *(required)* | `host:port`, e.g. `0.0.0.0:8080` |
| `path` | string | `/` | URL path that accepts `POST`; other paths return 404 |
| `auth_token` | string | `""` | when set, requires `Authorization: Bearer <token>` (constant-time compared); otherwise no auth |
| `max_body_bytes` | int | `1048576` (1 MiB) | hard cap on the request body — an oversized POST gets `413`, never an unbounded allocation |
| `discard_truncated_line` | bool | `true` | when the body cap trips **mid-line**, the last line arrives truncated. `true` (**default**) **drops** it so only whole lines are ever forwarded — no corrupted/partial log lines downstream (the over-cap line is lost entirely). Set `false` to instead forward the truncated prefix (at-least-once — a partial line may reach downstream). The tail beyond the cap is never forwarded either way |
| `tls_cert_file` | string | `""` | serve HTTPS (with `tls_key_file`); strongly recommended outside a lab |
| `tls_key_file` | string | `""` | server key (with `tls_cert_file`) |
| `tls_client_ca_file` | string | `""` | **mTLS**: require + verify client certificates against this CA (needs the server cert/key set) |
| `source` | string | `http_in` | `Event.Source` label stamped on every event |
| `facility` | int | `16` (local0) | syslog facility for posted events (they carry no PRI) |
| `severity` | int | `6` (info) | syslog severity for posted events |

A push-based ingress for sources that speak HTTP but can't run the agent — webhooks, IoT/edge devices, and app
log shippers — the HTTP analogue of `syslog_in`. `POST` a body of **newline-delimited log lines** to `path`;
each non-empty line becomes one event (message verbatim, `\r` trimmed, blank lines skipped). The input is kept
**vendor-neutral** — to structure a JSON body, add a [`parse_json`](#parse_json---message--structured-fields-nxlog-xm_json-parity) processor downstream rather
than baking a format into the input. Success returns `202 Accepted`; a non-`POST` returns `405`, a missing/bad
token `401`, and an oversized body `413`. Events enter the pipeline (spooled on backpressure — at-least-once
into the buffer); there is no application-level delivery ack back to the HTTP client (same contract as
`syslog_in`).

**Listener hardening:** alongside the body-size cap, the server bounds slow reads — a header timeout, a
whole-request read timeout, and a **5-minute idle keep-alive deadline** — so an unauthenticated peer that
dribbles a request body or opens an idle connection can't hold a handler open (slow-loris). Fixed defaults,
not configurable.

### `mqtt_in` ✅ — cross-platform (MQTT subscriber; IoT/edge ingest) — **Apex**

Subscribes to an MQTT broker and turns each published message into an event — the common IoT/edge shape
where sensors and devices publish telemetry to a broker rather than writing files or speaking syslog.
The agent dials **out** to the broker (nothing to open on the device side), subscribes to one or more
topic filters, and forwards payloads verbatim; add a `parse_json`/`parse_csv` processor downstream to
structure them.

| Option | Type | Default | Notes |
|---|---|---|---|
| `broker` | string | *(required)* | `host:port` of the MQTT broker to dial, e.g. `broker.local:1883` (or `:8883` with TLS) |
| `topics` | []string | *(required)* | one or more topic filters to subscribe to, e.g. `[sensors/#, devices/+/status]` |
| `qos` | int | `1` | subscription QoS: `0` (at-most-once) or `1` (at-least-once). **QoS 2 is not supported** — configuring `2` is rejected |
| `client_id` | string | `logrok-<hostname>-<pid>` | MQTT client identifier sent in CONNECT |
| `username` | string | `""` | optional CONNECT username; the password field is only sent alongside a username |
| `password` | string | `""` | optional CONNECT password (sent only when `username` is set) |
| `keep_alive` | duration | `60s` | PINGREQ interval; a link with no traffic for ~1.5× this is treated as dead and reconnected. Must be ≤ `65535s` (16-bit wire field) |
| `clean_session` | bool | `true` | CONNECT clean-session flag (`false` asks the broker to keep session state) |
| `max_message_bytes` | int | `1048576` (1 MiB) | hard cap on a single packet's declared remaining length — a hostile/oversized packet is rejected, never an unbounded allocation |
| `source` | string | `mqtt_in` | `Event.Source` label stamped on every event |
| `facility` | int | `16` (local0) | syslog facility for received messages (they carry no PRI) |
| `severity` | int | `6` (info) | syslog severity for received messages |
| `tls` | bool | `false` | enable TLS to the broker (also implied if any `tls_*` option below is set) |
| `tls_ca_file` | string | `""` | CA bundle to verify the broker certificate (empty = system roots) |
| `tls_cert_file` | string | `""` | client certificate for mutual TLS (with `tls_key_file`) |
| `tls_key_file` | string | `""` | client key for mutual TLS (with `tls_cert_file`) |
| `tls_insecure_skip_verify` | bool | `false` | **dev only** — skip broker certificate verification |

A pull-based ingress for the very common IoT/edge case where sensors and devices **publish to an MQTT broker**
rather than being able to run the agent. The agent dials OUT to the broker, subscribes to `topics`, and turns
each received `PUBLISH` into one event: the payload becomes `Event.Message` verbatim, `Host` is the broker
host, and `mqtt_topic` / `mqtt_qos` are recorded as fields. Like the other ingresses it is kept
**vendor-neutral** — to structure a JSON/CSV payload, add a [`parse_json`](#parse_json---message--structured-fields-nxlog-xm_json-parity)/[`parse_csv`](#parse_csv---csvdsv-message--structured-fields-nxlog-xm_csv-parity)
processor downstream rather than baking a format into the input. A QoS-1 message is acknowledged (PUBACK) only
**after the event is durable** — written to the disk spool, delivered, or deliberately dropped — not merely
accepted onto the pipeline. An agent crash at any point before that leaves the message un-acknowledged, so the
broker redelivers it; the agent never tells a broker it holds an event it could still lose. PUBACKs are emitted
in the order the messages arrived, so a slow event briefly holds back the acknowledgements behind it. On any
connection error the client reconnects with
jittered exponential backoff (cap 30s) and re-subscribes; on shutdown it sends a graceful DISCONNECT. The MQTT
3.1.1 client is hand-rolled from the stdlib (no new dependency, still cgo-free). **QoS 2 limitation:** a QoS-2
`PUBLISH` from the broker is emitted best-effort with a one-shot warning and **no** PUBREC handshake (the broker
may redeliver it) — subscribe at QoS 0 or 1 for defined semantics.

### `snmptrap_in` ✅ — cross-platform (SNMP trap receiver; OT/network devices) — **Apex**

Listens for SNMP **traps and informs** (UDP, IANA port 162) from network and OT devices — switches,
firewalls, UPSes, PLC gateways — and turns each notification into one event. Supports **v1**, **v2c**
and **v3 (USM** — per-user authentication and privacy**)** natively: no `snmptrapd` side-daemon, no
second config. An **InformRequest is acknowledged** on receipt, so senders using informs stop
retrying — and from there the event has the agent's usual durability (disk spool, at-least-once
forwarding).

| Option | Type | Default | Notes |
|---|---|---|---|
| `listen` | string | `0.0.0.0:162` | UDP listen address. Port 162 needs `CAP_NET_BIND_SERVICE` (or root); use a >1024 port and redirect if the agent runs unprivileged |
| `versions` | []string | `[v1, v2c]` | accepted SNMP versions, any of `v1`, `v2c`, `v3`. Adding `v3` requires `v3_users` |
| `community` | []string | `[]` (accept any) | v1/v2c community allow-list; a mismatching community is rejected and counted. Communities are plaintext on the wire — prefer v3 where devices support it |
| `v3_users` | list | `[]` | USM users for v3: each entry `{username, auth_protocol, auth_password, priv_protocol, priv_password}`. One user entry serves any number of sending engines with that user (keys are localized per sender engine automatically) |
| `v3_users[].username` | string | *(required)* | USM security name |
| `v3_users[].auth_protocol` | string | *(required)* | `sha256` (recommended), `sha`, `sha224`, `sha384`, `sha512`, or `md5` (legacy) |
| `v3_users[].auth_password` | string | *(required)* | authentication passphrase |
| `v3_users[].priv_protocol` | string | `""` (authNoPriv) | `aes` / `aes192` / `aes256` (recommended) or `des` (legacy); empty = authenticated but unencrypted |
| `v3_users[].priv_password` | string | `""` | privacy passphrase (required when `priv_protocol` is set) |
| `oid_names` | map | `{}` | optional OID → name labels, e.g. `{"1.3.6.1.6.3.1.1.5.3": linkDown}` — a matching trap gains a `trap_name` field and varbind keys use the name. No MIB compiler needed; full MIB semantics stay downstream |
| `queue_size` | int | `1024` | bounded hand-off queue between the UDP listener and the pipeline; overflow during a trap storm is counted and dropped (UDP cannot be back-pressured) |
| `source` | string | `snmptrap_in` | `Event.Source` label stamped on every event |
| `facility` | int | `23` (local7) | syslog facility for received traps |
| `severity` | int | `5` (notice) | syslog severity for received traps |

Each trap becomes one event: `Host` is the sender (for v1, the trap's agent-address field — relays
preserve the true origin), `Message` is the trap OID (or its `oid_names` label) plus a short varbind
summary, and every varbind lands as a `vb_<oid>` field with the value rendered to a stable string
(integers decimal, strings UTF-8 or hex, OIDs dotted). v1 traps are translated to v2c trap OIDs the
standard way (RFC 2576), so a mixed v1/v2c estate produces uniform fields. `sysuptime`, the
`snmp_version`, and (for informs) `inform: true` ride along as fields. Traps are UDP: delivery to
the agent has no protocol guarantee (use informs where the device supports them — those are
acknowledged); once accepted, the pipeline's usual durability applies. Verified against the
net-snmp reference tools across the v3 auth/priv matrix in the test battery.

### `modbus_in` ✅ — cross-platform (Modbus TCP; OT/PLC register polling) — **Apex**

Polls **Modbus TCP** devices — PLCs, RTUs, drives, meters, and the serial-to-TCP gateways that front
them — and emits **events, not register streams**. Modbus registers are time-series telemetry; a
1 Hz poll of fifty registers is millions of rows a day that no SIEM rule will ever read. This input
watches the registers you name and forwards only what a log platform can act on: a value that
**changed**, a value that **crossed a bound**, and a device that **stopped answering**. A quiet plant
produces almost no events, which is the point.

**Read-only by construction.** Only the four Modbus read function codes (`0x01`–`0x04`) exist in the
agent. The write codes are not implemented and not merely disabled by configuration — there is no code
path through which the agent can command a PLC, and an automated check fails the build if one is ever
added.

**Two transports.** `transport: tcp` (the default) speaks Modbus TCP to a device or gateway.
`transport: rtu` speaks Modbus RTU over a serial line — an RS-485/RS-232 port on the collecting
host — for equipment that has no Ethernet at all. Everything above the wire is identical: the same
watches, the same events, the same read-only guarantee.

| Option | Type | Default | Notes |
|---|---|---|---|
| `targets` | list | *(required)* | the devices to poll; at least one. Each device is polled independently, on its own connection and schedule |
| `targets[].transport` | string | `tcp` | `tcp` for Modbus TCP, `rtu` for Modbus RTU over a serial line. Chooses which of `address`/`device` is required; setting the wrong one is a config error rather than a silent no-op |
| `targets[].address` | string | *(required for `tcp`)* | device `host:port` (Modbus TCP is usually port 502) |
| `targets[].device` | string | *(required for `rtu`)* | serial device — e.g. `/dev/ttyUSB0`, `/dev/ttyS0`, or `COM3` on Windows |
| `targets[].baud` | int | `19200` | line rate: 1200, 2400, 4800, 9600, 19200, 38400, 57600 or 115200. The Modbus default is 19200 |
| `targets[].data_bits` | int | `8` | must be 8. Modbus RTU is a binary encoding; 7 data bits is Modbus ASCII, which this input does not implement |
| `targets[].parity` | string | `even` | `even` (the Modbus default), `odd`, or `none` |
| `targets[].stop_bits` | int | `1`, or `2` when `parity: none` | the specification keeps the character frame 11 bits wide, so a line without a parity bit uses two stop bits. Override only if your device disagrees |
| `targets[].unit_id` | int | `1` | Modbus unit / slave id (0–255). Gateways use it to select the device behind them |
| `targets[].poll_every` | duration | `5s` | how often to poll this device. Floor of `100ms` — faster is register streaming, which this input deliberately does not do |
| `targets[].timeout` | duration | `3s` | per-request dial/read timeout. Must not exceed `poll_every`, or polls would overlap on a slow device |
| `targets[].address_base` | string | `protocol` | how register numbers are written. `protocol` = the zero-based address that goes on the wire. `data_model` = the 5-digit vendor notation (`4xxxx` holding, `3xxxx` input, `1xxxx` discrete, `0xxxx` coil), so `holding:40012` means the twelfth holding register. See the addressing note below |
| `targets[].max_read_gap` | int | `8` | how many unwatched registers may be read to bridge two watches into one request. Higher = fewer requests, more registers read; `0` = never bridge |
| `targets[].watches` | list | *(required)* | the registers to observe on this device; at least one |
| `targets[].watches[].name` | string | *(required)* | the watch's name; unique per device, and the key every event carries |
| `targets[].watches[].register` | string | *(required)* | `<kind>:<address>` where kind is `coil`, `discrete`, `input` or `holding` — e.g. `holding:11`, `coil:12` |
| `targets[].watches[].type` | string | `bool` for bit kinds, `int16` for word kinds | how to read the value: `bool`, `int16`, `uint16`, `int32`, `uint32`, `float32`. The 32-bit types read two consecutive registers |
| `targets[].watches[].scale` | float | `1` | raw value × scale = the engineering value used in events and threshold tests (e.g. `0.1` for a tenths-of-a-bar register) |
| `targets[].watches[].word_order` | string | `big` | register order for 32-bit types: `big` (high word first) or `little` (the word swap many devices use). Rejected on 16-bit types |
| `targets[].watches[].on_change` | bool | `false` | emit a `state_change` event whenever this value changes |
| `targets[].watches[].deadband` | float | `0` | with `on_change`, suppress changes smaller than this — an analog-noise guard. The reference value does **not** advance while inside the band, so a slow drift is still reported once it accumulates past it. Needs `on_change` |
| `targets[].watches[].threshold` | map | *(none)* | bounds on a numeric value: `{above: <n>}`, `{below: <n>}`, or both. Crossing a bound emits a `threshold` event, and returning inside emits another |
| `targets[].watches[].threshold.above` | float | *(none)* | emit when the value goes strictly **above** this |
| `targets[].watches[].threshold.below` | float | *(none)* | emit when the value goes strictly **below** this |
| `snapshot_every` | duration | `0` (off) | optional slow-cadence baseline: one `snapshot` event per device carrying every watch's current value, for audit and trend anchoring. Minimum `1m` — a snapshot is a baseline, not a stream |
| `emit_initial_snapshot` | bool | `false` | with `snapshot_every` set, also emit a snapshot on the first successful poll (a fleet restart then re-baselines immediately) |
| `failures_before_unavailable` | int | `2` | consecutive failed polls before an `availability` event is emitted, so one dropped packet is not an outage |
| `queue_size` | int | `256` | bounded hand-off queue between the pollers and the pipeline; overflow is counted and dropped rather than stalling every device's schedule |
| `source` | string | `modbus_in` | `Event.Source` label stamped on every event |
| `facility` | int | `16` (local0) | syslog facility for emitted events |
| `severity` | int | `5` (notice) | baseline syslog severity. Threshold entry raises to `4` (warning) and a device that stops answering to `3` (error) |

**Events.** Every event carries `modbus.event` (the kind), `modbus.address` (the `host:port` or the
device path), `modbus.transport` (`tcp` or `rtu`), and `modbus.unit_id`;
`Host` is the device. The four kinds, carried in the `modbus.event` field:

- **state_change** — a watched value changed (past its `deadband`, if set).
  Adds `modbus.watch`, `modbus.register`, `modbus.old`, `modbus.new`.
- **threshold** — a watched value crossed a bound, or came back inside.
  Adds `modbus.watch`, `modbus.register`, `modbus.value`, `modbus.bound` (`above` or `below`),
  `modbus.limit`, and `modbus.state` (`entered` or `cleared`).
- **availability** — the device stopped answering reads, or started again.
  Adds `modbus.state` (`down` or `up`), `modbus.fault` (`transport` = unreachable, `protocol` = the
  device answered but refused the read), and `modbus.reason`.
- **snapshot** — the `snapshot_every` cadence elapsed.
  Adds one `modbus.value.<watch>` field per watch.

**The first poll is silent.** A successful first poll records the device's state and emits nothing, so
restarting the agent does not report every register in the plant as having just changed. Threshold
state is seeded the same way: a value already out of bounds at startup does not re-alarm, but coming
back inside is still reported.

**Register addressing — the one thing to get right.** Vendor manuals are written in the 5-digit *data
model* notation (`40012`), while the wire uses a zero-based *protocol* address (`11` for that same
register). Both are supported, but you must say which you are writing: leave `address_base` at
`protocol` and give protocol addresses, or set it to `data_model` and paste the vendor's numbers
unchanged. If a protocol-base config contains a number sitting in its own kind's data-model band
(`holding:40012`, say), the agent logs a warning naming the fix at startup — it cannot reject it,
because that is also a legal protocol address.

**Politeness.** OT devices are frequently single-threaded and fragile. Each device gets one connection
with one request in flight at a time; adjacent watches are coalesced into as few reads as the protocol's
per-request ceilings allow (a run of eight adjacent registers costs one read, not eight); and
`poll_every` has a hard floor. A device that stops answering is retried on the normal schedule, not
hammered.

**Serial lines (`transport: rtu`).** A few things differ from TCP, all of them inherent to serial:

- **One device at a time per line.** Each configured target owns its serial port exclusively — that
  is what RS-485 multi-drop requires, since two masters on one line corrupt each other's frames. To
  poll several unit ids on one physical line, that is a single target per unit id only if they are
  on *different* ports; a shared line with several units is not supported today.
- **Integrity rests entirely on the CRC.** RTU frames carry no length field and no transport
  checksum underneath, so a corrupted frame is caught by its CRC-16 and the poll is failed rather
  than a wrong value being reported. Persistent CRC failures raise the same `availability` event as
  an unreachable device, with the reason naming the CRC.
- **The line is resynchronised before every request**, so one bad exchange cannot leave every later
  reply shifted by a frame.
- **Permissions.** The agent must be able to open the device. On Linux that usually means adding its
  service user to the `dialout` group; a permission failure is reported at startup, not silently.

**Optional build.** `modbus_in` ships in the standard binary. A reduced-surface build that omits it
entirely (smaller SBOM and attack surface) is available on request for regulated deployments.

### `journald` ✅ *(Linux only)*
Reads the systemd journal by following `journalctl -o json` as a subprocess (keeps the binary static + cgo-free
— an internal architecture decision). On non-Linux builds this is a stub that errors if started. The JSON field
mapping is **verified against `journalctl -o json` on a live systemd host**; the `cursor_file` resume cycle
(stop → entries land while down → restart → caught up, nothing re-delivered) is **integration-tested against a
real journal** on every CI run.

| Option | Type | Default | Notes |
|---|---|---|---|
| `units` | []string | all | restrict to these units (`-u`), e.g. `[nginx.service]` |
| `identifiers` | []string | all | restrict to these `SYSLOG_IDENTIFIER` tags (`-t`, e.g. `[myapp]`) — matches `logger`/syslog(3) entries, which carry no unit |
| `priority` | string | all | max priority (`-p`): `0`–`7` or a name like `warning` |
| `cursor_file` | string | `""` | **agent-owned** resume checkpoint: the last forwarded entry's `__CURSOR`, saved atomically (every 100 entries + on shutdown) and fed back as `--after-cursor` on restart. Empty = follow new entries only (`-n0`) |

Maps `PRIORITY`→severity, `SYSLOG_FACILITY`→facility (default local0/16), `_HOSTNAME`→host,
`_SYSTEMD_UNIT`/`SYSLOG_IDENTIFIER`→`journald:<unit>` source, `__REALTIME_TIMESTAMP`→timestamp, plus
`unit`/`pid`/`uid`/`syslog_identifier`/`boot_id` fields (`uid` = the journal's kernel-verified `_UID` sender —
the user-traceability field compliance mappings reference). Resume via `cursor_file` is bounded-dup at the crash boundary
(checkpoint trails the forwarded send — a crash replays the unsaved tail, never skips it; no entry ack — see
the at-least-once note under [`buffer`](#buffer)). The checkpoint is deliberately **not** journalctl's `--cursor-file`: in follow mode
journalctl only persists that on a graceful exit it never gets under service management (verified on
systemd 249 — nothing saved under SIGTERM/SIGINT/SIGKILL), which would have made every resume silently lossy.

### `linux_audit` ✅ *(Linux only)* — **Apex**

Collects **Linux kernel audit records** — the security trail the kernel's audit subsystem emits and auditd
persists: process executions with full command lines (execve rules), file-access watches, logins, privilege
changes (the NXLog `im_linuxaudit` counterpart). Reach for this instead of a plain `filetail` on
`audit.log` because audit events span *several* raw records sharing one serial: this input **reassembles
them into one event per kernel audit event**, hex-decodes EXECVE arguments and PROCTITLE, joins `argv`,
and indexes multiple PATH records — the structure a downstream detection actually needs. On non-Linux
builds this is a stub that validates config but errors if started.

| Option | Type | Default | Notes |
|---|---|---|---|
| `mode` | string | `file` | `file` follows the auditd-written log; `multicast` joins the kernel's read-only audit netlink group (see the tradeoff below) |
| `path` | string | `/var/log/audit/audit.log` | file mode: the audit log path (glob allowed; with several matches the most recently modified wins). Invalid in multicast mode |
| `read_from` | string | `end` | file mode: `end` (new records only) or `beginning` (whole file). Invalid in multicast mode |
| `reassembly_timeout` | duration | `2s` | flush an audit event whose `EOE` terminator never arrives (EOE isn't guaranteed in file mode; single-record `USER_*` events have none) |
| `max_in_flight` | int | `1024` | bound on concurrently-buffered audit events (serials) during reassembly; overflow emits the oldest event as-is with a counted warning — memory never grows unbounded |
| `source` | string | `linux_audit` | `Event.Source` label |

**Mode tradeoff.** `file` works on every distro running auditd and needs only read permission on
`audit.log` (root or the `adm`/`root` group, distro-dependent) — use it by default. `multicast` binds a
`NETLINK_AUDIT` socket and joins the **`AUDIT_NLGRP_READONLY`** multicast group (kernel ≥ 3.16): a live
kernel feed with no file I/O and no dependency on auditd's log format/rotation settings — but it requires
**`CAP_AUDIT_READ`** (typically root; without it the input fails at start with an explicit "requires
CAP_AUDIT_READ … or use mode: file" error), and records the kernel emitted while the agent was down are
missed (the file has them). Multicast is **strictly passive**: the agent never registers as the audit
daemon (never calls `audit_set_pid`), so a running auditd keeps working, and keeps writing `audit.log`,
untouched. Neither mode loads audit *rules* — configure those with `auditctl`/`augenrules` as usual; with
no rules loaded there is simply no traffic.

**Checkpointing (documented v1 decision).** File mode persists **no offset checkpoint**: a restart
re-follows per `read_from` (default: from the end). Events that land during an agent restart stay in
`audit.log` itself, so the recovery story is replay-from-file; a byte-offset checkpoint is the planned
follow-up.

**Decoding (honest subset).** EXECVE `a0…aN` arguments (including chunked `aN[k]` long args) are
hex-decoded where the kernel hex-encoded them and joined into `argv`; `PROCTITLE` is hex-decoded
(NUL separators → spaces); PATH records land as `path`, `path.2`, … (`path_mode`, `path.2_mode`, … for
their per-record attributes); `CWD` → `cwd`; the nested `msg='…'` payload of `USER_*` records is
flattened; enriched-format (`log_format = ENRICHED`) resolution fields parse as ordinary fields.
**`syscall` deliberately stays numeric** — name resolution is arch-dependent (the `arch=c000003e` vs
`b7` tables differ), so resolve it downstream. `SOCKADDR`'s `saddr` passes through as raw hex. A field
name already set by an earlier record is preserved by namespacing the newcomer as `<rectype>_<key>`
(then `.2`, `.3`, …) — later records never overwrite earlier ones.

The **message** is the decoded `argv` for events containing EXECVE (the most useful one-line rendering of
a process execution); otherwise the primary record's raw line (the first SYSCALL record, else the first
record). Every event carries `audit_serial` and `audit_types` (comma list of the record types reassembled
into it). Timestamp comes from the audit header (`SECS.MILLIS`), severity is notice (5), facility is 13
("log audit"), `host` is the agent's hostname. Malformed lines are skipped and counted, never fatal.

```yaml
inputs:
  - type: linux_audit          # file mode: follow auditd's log (default)
    # path: /var/log/audit/audit.log
    # read_from: end

  - type: linux_audit          # OR: live kernel feed, coexists with auditd
    mode: multicast            # requires CAP_AUDIT_READ (typically root)
```

### `oslog` ✅ *(macOS only)*
Reads the macOS unified log (OSLog) by running `/usr/bin/log show|stream --style ndjson` as a subprocess
(keeps the binary static + cgo-free — an internal architecture decision). On non-macOS builds this is a stub that
errors if started. On startup it **backfills** the gap since the last run via `log show --start <checkpoint>`,
then follows the live log via `log stream`. The `log` binary is always invoked by absolute path
(`/usr/bin/log`) because an interactive shell has a `log` builtin that would otherwise shadow it.

| Option | Type | Default | Notes |
|---|---|---|---|
| `predicate` | string | `""` | optional NSPredicate passed verbatim to `--predicate`, e.g. `subsystem == "com.apple.securityd"` |
| `level` | string | `default` | `default` \| `info` \| `debug` — widen captured verbosity (`--level` for stream, `--info`/`--debug` for the backfill) |
| `checkpoint_file` | string | `""` | **agent-owned** resume checkpoint: the last forwarded event's `timestamp`, saved atomically (every 100 events + on shutdown) and fed back as `log show --start` on restart. Empty = live stream only, no backfill |
| `backfill_on_start` | bool | `true` | when a checkpoint exists, `log show` the gap before streaming; set `false` to stream live only |

Maps `messageType`→severity (`Fault`→crit/2, `Error`→err/3, `Default`→notice/5, `Info`→info/6, `Debug`→debug/7),
`eventMessage`→message, `processImagePath` basename→`oslog:<process>` source, `timestamp`→timestamp, plus
`process`/`process_image_path`/`pid`/`subsystem`/`category`/`sender_image_path`/`thread_id`/`trace_id`/`activity_id`/`message_type`/`format_string`
fields (`format_string` is the un-interpolated OSLog message template — the natural key for
event-template grouping/dedup; `pid` also rides in RFC 5424 PROCID, see [`syslog` output](#syslog---rfc-5424-over-tcp-tlsmtls-or-udp-diode-mode)). The unified log carries no hostname, so `host` is the agent's `os.Hostname()`. Resume via
`checkpoint_file` is bounded-dup at the boundary (`--start` is inclusive; the checkpoint trails the forwarded
send — a crash replays the unsaved tail, never skips it; same at-least-once class as journald).

---

## `processors`

Run in order; a processor returning *drop* removes the event.

### `add_fields` ✅

Stamps static key/value fields onto every event — tenant, site, environment, device type. Use it to
label a fleet at the edge so events stay attributable downstream without receiver-side lookups; the
added fields travel like any other event field (RFC 5424 structured data on the `syslog` output).

| Option | Type | Notes |
|---|---|---|
| `fields` | map | static key/value fields added to every event (values stringified) |

### `filter` ✅

Drops events you don't want to forward, by severity ceiling or exact field values — the simplest way to
cut noise at the edge (e.g. drop info/debug, keep warnings and worse; or exclude a known-chatty unit).
For conditions these keys can't express, use the `expr` processor; for rate-based thinning rather than
hard drops, see the reduction processors below.

| Option | Type | Notes |
|---|---|---|
| `max_severity` | int | keep only events with `Severity <= N` (syslog: 0=emerg … 7=debug), i.e. drop less-severe |
| `exclude_fields` | map[string][]string | drop if `Fields[key]` equals any listed value |
| `include_fields` | map[string][]string | if set, **keep only** events where some `Fields[key]` matches |

### `parse_csv` ✅ — CSV/DSV message → structured fields (NXLog `xm_csv` parity)

For sources that emit delimited rows — W3C/IIS logs, exported audit tables, TSV/DSV app logs: parses the
message (or any field) against a fixed column list and turns each column into a queryable field at the
edge, instead of shipping one opaque line for the receiver to untangle.

| Option | Type | Default | Notes |
|---|---|---|---|
| `columns` | []string | *(required)* | field name per position; `"-"` skips a column (W3C/IIS style) |
| `separator` | string | `,` | single character; `"\t"` for TSV, `" "` for W3C/IIS logs |
| `source` | string | `message` | what to parse: the event message, or any field name |
| `prefix` | string | `""` | prepended to created field keys |
| `on_error` | string | `keep` | `keep` (non-matching rows pass through untouched) or `drop`. A row whose value count doesn't match `columns` is an error — partial application would scatter misaligned fields |

RFC 4180 quoting handled (embedded separators/quotes); a `"-"` **value** is the W3C null and creates no
field. With `separator: " "` and `"-"` placeholder columns this parses W3C/IIS-shaped logs directly.

### `parse_json` ✅ — message → structured fields (NXLog `xm_json` parity)

For applications that write JSON lines — into a file, a container stream, or syslog: explodes the JSON
object in the message (or any field) into structured fields at the edge, so values are queryable
downstream instead of buried in a text blob. Safe on mixed text/JSON streams — by default a non-JSON
line passes through untouched.

| Option | Type | Default | Notes |
|---|---|---|---|
| `source` | string | `message` | what to parse: the event message, or any field name |
| `prefix` | string | `""` | prepended to every created field key, e.g. `json.` (collision control) |
| `drop_message` | bool | `false` | blank the message after a **successful** parse (failed parses keep it) |
| `on_error` | string | `keep` | `keep` (pass non-JSON events through untouched — safe for mixed text/JSON streams) or `drop` |

Parses a JSON **object** into fields: nested objects flatten with dots (`http.resp.status`), arrays stay as
compact JSON strings, number literals are preserved verbatim (64-bit ids survive — no float64 round-trip),
`null` values are skipped, parsed keys overwrite same-name fields (use `prefix` to avoid). Top-level
arrays/scalars and malformed JSON take the `on_error` path. Nesting is capped (32 levels) — deeper subtrees
are stringified, never a stack risk.

### `parse_kv` ✅ — key=value message → structured fields (NXLog `xm_kv` parity)

For `key=value`-shaped logs — space-delimited app logs and, with custom separators, Cisco ASA, CEF
extensions, and query strings: splits the pairs in the message (or any field) into structured fields at
the edge so each value is queryable on its own.

| Option | Type | Default | Notes |
|---|---|---|---|
| `source` | string | `message` | what to parse: the event message, or any field name |
| `prefix` | string | `""` | prepended to every created field key |
| `kv_separator` | string | `=` | single character between a key and its value |
| `pair_separator` | string | `" "` (space) | single character between pairs |
| `drop_message` | bool | `false` | blank the message after a **successful** parse (only when `source` is the message) |
| `on_error` | string | `keep` | `keep` (pass through when no pairs are found) or `drop` |

Splits `key=value` pairs into fields. A `pair_separator` inside a double-quoted value is not a delimiter
(`msg="hello world"` stays one pair); each token splits on the **first** `kv_separator` (so `url=http://x?a=b`
keeps its value intact); surrounding double-quotes are stripped from values. Tokens with no separator or an
empty key are skipped (KV streams carry noise). Zero parsed pairs takes the `on_error` path. Repeated keys:
last value wins (use `prefix` to scope). Covers space-delimited app logs and, with custom separators, the
Cisco ASA / CEF-extension / query-string shapes.

### `parse_xml` ✅ — XML message → structured fields (NXLog `xm_xml` parity)

For XML payloads — e.g. raw Windows event XML forwarded with `windows_eventlog`'s `render: xml`, or
XML-emitting devices and applications: flattens the document in the message (or any field) into
dotted-path fields so element values and attributes are queryable downstream.

| Option | Type | Default | Notes |
|---|---|---|---|
| `source` | string | `message` | what to parse: the event message, or any field name |
| `prefix` | string | `""` | prepended to every created field key, e.g. `xml.` |
| `drop_message` | bool | `false` | blank the message after a **successful** parse (only when `source` is the message) |
| `on_error` | string | `keep` | `keep` (pass non-XML events through untouched) or `drop` |

Flattens an XML document into fields: nested elements join with dots (`event.system.level`), element text is
the value at that path, attributes emit as `path@attr` (`event@id`), and repeated sibling elements are
disambiguated with `.2`, `.3`, … (`event.data`, `event.data.2`). The parse is **atomic** — malformed XML or a
document with no elements writes no fields and takes the `on_error` path (no partial scatter). Nesting is
capped (32 levels). Pure stdlib `encoding/xml`, no cgo.

### `expr` ✅ — minimal safe expression (set/drop/rename + conditions)
An ordered list of `rules`. Each rule has an optional boolean condition (`if`); when it matches, the rule either
drops the event or mutates its fields. Rules run in order; a `drop` short-circuits the rest. Within a rule the
order is **set → remove → rename**.

| Option | Type | Notes |
|---|---|---|
| `rules[].if` | string | condition; empty = always. Grammar below. |
| `rules[].drop` | bool | drop the event when the condition matches |
| `rules[].set` | map[string]string | set/overwrite these fields |
| `rules[].remove` | []string | delete these fields |
| `rules[].rename` | map[string]string | rename `old: new` (no-op if `old` is absent) |

**Condition grammar.** Operands: built-ins `severity`, `facility` (numeric), `message`, `host`, `source`
(string), and `fields.<key>` (dotted keys allowed, e.g. `fields.device.type`); number/`"string"` literals.
Operators: `== != < <= > >=`, the string ops `contains startswith endswith matches` (`matches` takes a regex
**literal**), boolean `&& || !`, and parentheses. A bare operand tests truthiness (non-empty / non-zero). `<`-family
compares numerically (non-numeric → false); `==`/`!=` compare numerically if both sides are numbers, else as strings.

```yaml
- type: expr
  rules:
    - if: 'severity >= 5'            # drop debug/info noise (syslog: higher = less severe)
      drop: true
    - if: 'fields.app == "nginx"'
      set:    { tier: frontend }
    - if: 'message contains "password"'
      remove: [raw]
```

🔜 `multiline`, format auto-detect as standalone processors (format-detect already handled inside `syslog_in`)
(planned).

### Reduction processors ✅

Cut log volume on the host **before** forwarding — thin a noisy source, deduplicate error storms, or trim
oversized fields. All three stateful processors (`sample`, `throttle`, `dedup`) support a **safety guard**
that bypasses reduction for security-critical events:

- **`keep_min_severity`** (int) — events with `Severity <= N` always pass untouched (syslog: 0=emerg … 3=err
  … 7=debug; lower number = higher severity). Leave unset to apply reduction to all severities.
- **`keep_if`** (string, `expr` grammar) — if the condition matches, the event bypasses reduction regardless
  of severity. Uses `&&`/`||` operators. Example: `'severity <= 3 && fields.event_id == "4624"'`.

Drop attribution is tracked per processor: scrape `logrok_agent_processor_dropped_total{processor="<name>"}`.
When `suppress_count` is enabled (default on), the next event that passes after a suppressed run carries a
`suppress_count` field with the number of events that were dropped.

#### `sample` ✅ — 1-in-N probabilistic sampler — **Apex**

Keeps 1 event in N, where N = round(1/`ratio`). Without a `key`, uses a counter (sequential); with a `key`,
hashes the key for consistent sampling (the same key value always gets the same decision). The effective
keep rate is ~1/N but is approximate — it depends on hash distribution and can deviate over skewed key
populations.

**`key` field names:** the built-in names are `host`, `source`, `message`, `severity`, and `facility`; any
other name is matched against event `Fields` at runtime — a misspelled name yields an empty value and
silently collapses unrelated events into the same bucket. Use `"*"` to key on the full event (all fields),
but `"*"` must be the only entry in the list — mixing it with other fields is a configuration error.

| Option | Type | Default | Notes |
|---|---|---|---|
| `ratio` | float | *(required)* | Fraction of events to keep: `0 < ratio <= 1`. `ratio: 0.1` keeps ~1-in-10; `ratio: 1.0` keeps all |
| `key` | []string | `[]` | Field names to build a hash key for consistent sampling. Omit for sequential counter mode. `"*"` alone = full event key |
| `keep_min_severity` | int | *(none)* | Bypass sampling for events with `Severity <= N` (security-critical guard) |
| `keep_if` | string | `""` | `expr`-grammar condition; matching events bypass sampling. Uses `&&`/`||` |

```yaml
- type: sample
  ratio: 0.1                   # keep ~10% of events (1-in-10)
  key: [host, source]          # consistent: same host+source always kept or always dropped
  keep_min_severity: 3         # errors and above always pass
  keep_if: 'fields.event_id == "4625"'   # account-lockout events always pass
```

#### `quota` ✅ — hard daily volume cap — **Apex**

Caps a pipeline's daily volume: a per-UTC-day budget in events and/or approximate bytes; events
over budget are **dropped and counted** until the day rolls over. The blunt instrument for "this
source must never cost more than X/day" — pair with [`sample`](#sample--)/[`throttle`](#throttle--)
for shapelier reduction. Exempt events (`keep_min_severity` / `keep_if`) always pass and consume
no budget — a volume cap must never swallow page-worthy events.

| Option | Type | Default | Notes |
|---|---|---|---|
| `max_events` | int | `0` (off) | events admitted per UTC day; at least one of the two budgets must be > 0 |
| `max_bytes` | int | `0` (off) | approximate bytes admitted per UTC day (pre-encoding: message + fields + host + source — a dial, not an invoice) |
| `suppress_count` | bool | `true` | the first event admitted after budget-drops carries `quota_dropped` / `quota_dropped_bytes` fields, so the gap is visible downstream |
| `keep_min_severity` | int | *(off)* | events at or above this urgency (numerically ≤, syslog order) bypass the budget entirely |
| `keep_if` | string | `""` | expression (the filter grammar); matching events bypass the budget |

Two honest caveats, by design: the window is the **UTC calendar day** (tumbling), and the spent
budget is **in-memory — an agent restart refills it**. Treat the cap as an operating dial, not a
billing guarantee.

### `throttle` ✅ — per-key rate cap — **Apex**

Allows at most `max` events per `window` per key (or globally when `key` is omitted). Excess events in the
window are dropped. The next event that passes after a suppressed window carries `suppress_count` (by default).

The window is a **fixed (tumbling) window** anchored to the first event seen for each key. It does not slide.
A burst straddling a window boundary can briefly allow up to ~2×`max` events before the next window resets
the count. Omitting `key` uses a single global bucket across all events.

| Option | Type | Default | Notes |
|---|---|---|---|
| `max` | int | *(required)* | Maximum events to pass per window |
| `window` | duration | *(required)* | Window duration, e.g. `1m`, `5m`, `1h` |
| `key` | []string | `[]` | Field names for per-key buckets. Omit for a single global bucket |
| `max_keys` | int | `50000` | Cap on the number of concurrent key buckets (fail-open on overflow — evicts the oldest key, which then gets a fresh allowance) |
| `suppress_count` | bool | `true` | Attach a `suppress_count` field to the next passing event after a suppressed run |
| `keep_min_severity` | int | *(none)* | Bypass throttling for events with `Severity <= N` |
| `keep_if` | string | `""` | `expr`-grammar condition; matching events bypass throttling. Uses `&&`/`||` |

```yaml
- type: throttle
  max: 10
  window: 1m
  key: [host, source]          # per host+source rate cap
  max_keys: 10000
  suppress_count: true         # default; next passing event gets suppress_count=N
  keep_min_severity: 3
  keep_if: 'fields.event_id == "4625" && severity <= 3'
```

#### `adaptive_sample` ✅ — hold volume near a target by adjusting the sample rate — **Apex**

Measures how many events a window carried and picks a sampling rate for the next one, so output
tracks a target as input rises and falls. Never mutates the message.

**Why not `sample` or `throttle`?** They solve different problems:

| | behaviour | weakness |
|---|---|---|
| `sample` | fixed 1-in-N | output still swings with input — a burst still floods the receiver |
| `throttle` | hard cap per window | admits whatever arrived **first**, then goes silent — the tail of an incident is invisible |
| `adaptive_sample` | `target` free, then a sampled tail | a **bound**, not a precise cap (see below) |

**It is a bound, not a hard ceiling.** Each window admits `target` events unsampled and then a sampled
tail, so a 10x burst lands near **2x** target rather than 10x. That is deliberate: it keeps both the head
*and* the tail of a burst visible, and it stops a stale rate from starving a quiet window that follows a
noisy one. If you need a hard ceiling, use `throttle`.

**Every admitted event carries the effective rate** (`sample_rate` by default) so counts survive sampling:
20 events stamped rate 50 means roughly 1000 occurred. Without it, sampling silently corrupts every
downstream count — which is why the annotation is on by default. Set `annotate: ""` to disable it.

The first window after startup admits everything: with no measurement there is no basis for discarding.

| Option | Type | Default | Notes |
|---|---|---|---|
| `target` | int | *(required)* | Events to admit per window before sampling begins. Must be > 0 |
| `window` | duration | `60s` | Measurement/adjustment window |
| `key` | []string | `[]` | Sample each key independently, so a noisy host can't sample a quiet one away |
| `max_keys` | int | `50000` | Key-cache bound; the oldest key is evicted and restarts at admit-all (fails open) |
| `annotate` | string | `sample_rate` | Field stamped with the effective 1-in-N. `""` disables |
| `keep_min_severity` | int | *(unset)* | Never sample events at or below this severity |
| `keep_if` | string | `""` | Expression; matching events bypass sampling entirely |

```yaml
- type: adaptive_sample
  target: 1000            # ~1000 events/window before sampling kicks in
  window: 60s
  key: [host]             # per-host budgets
  keep_min_severity: 3    # never sample warnings and worse
```

#### `dedup` ✅ — per-key window deduplication — **Apex**

Keeps the first event per key per window; identical repeats within the window are dropped. The default key
is `[host, message]` (60s window). The special key `["*"]` treats the full event (all fields) as the
identity key. The next event that passes after a suppressed window carries `suppress_count` (by default).

| Option | Type | Default | Notes |
|---|---|---|---|
| `key` | []string | `[host, message]` | Field names for the identity key. `["*"]` = full event (all fields, order-independent) |
| `window` | duration | `60s` | How long a key is suppressed after it is first seen |
| `max_keys` | int | `50000` | Cap on the number of concurrent key entries (fail-open on overflow — evicts oldest, which then gets a fresh window) |
| `suppress_count` | bool | `true` | Attach a `suppress_count` field to the first passing event after a suppressed run |
| `keep_min_severity` | int | *(none)* | Bypass deduplication for events with `Severity <= N` |
| `keep_if` | string | `""` | `expr`-grammar condition; matching events bypass deduplication. Uses `&&`/`||` |

```yaml
- type: dedup
  key: [host, message]         # default; suppress identical host+message pairs
  window: 60s                  # default
  max_keys: 50000              # default
  suppress_count: true         # default; next passing event gets suppress_count=N
  keep_min_severity: 3
```

Using `["*"]` to deduplicate by exact event identity (all fields must match):

```yaml
- type: dedup
  key: ["*"]
  window: 5m
```

#### `trim_fields` ✅ — field drop and value truncation — **Apex**

Shrinks event bytes by removing unwanted fields and/or truncating long values. Never drops events. `keep_only`
is applied first; then `drop`/`drop_matching`; then truncation of remaining values. Truncated values are
suffixed with `…`.

**Truncation is not a hard byte cap on the output.** `max_message_bytes` and `max_field_bytes` cap the
retained *content* bytes; the 3-byte UTF-8 `…` marker is appended afterwards, so a truncated value can be
up to N+3 bytes. Size limits at your SIEM or syslog receiver must account for this.

| Option | Type | Default | Notes |
|---|---|---|---|
| `drop` | []string | `[]` | Field names to unconditionally remove |
| `drop_matching` | string | `""` | Regexp; remove all fields whose name matches |
| `keep_only` | []string | `[]` | Allowlist: remove **all** fields **not** in this list (applied before `drop`/`drop_matching`) |
| `max_message_bytes` | int | `0` | Truncate the message to at most this many content bytes (rune-safe, appends `…`; output ≤ N+3 bytes). `0` = no limit |
| `max_field_bytes` | int | `0` | Truncate every field value to at most this many content bytes (rune-safe, appends `…`; output ≤ N+3 bytes). `0` = no limit |

```yaml
- type: trim_fields
  drop: [thread_id, stack_hash]
  drop_matching: '^debug_'     # remove any field whose name starts with debug_
  keep_only: []                # omit (or leave empty) to skip the allowlist step
  max_message_bytes: 4096
  max_field_bytes: 512
```

#### `redact` ✅ — remove sensitive values before they leave the host — **Apex**

Rewrites matching substrings in the message and/or field values. Never drops events.

Redaction happens **on the endpoint that produced the data**, so a value you must not retain never crosses
the wire. Masking at the collector still means the secret travelled and was written down somewhere upstream.

At least one of `patterns` or `regex` is required — a `redact` step that matches nothing would pass sensitive
data through while looking configured, so the agent refuses to start instead.

**Built-in patterns are deliberately conservative.** A pattern that fires on the wrong value destroys data
nobody can recover or even identify downstream, so `credit_card` is validated with the Luhn checksum rather
than matching any run of digits, `ipv4` range-checks each octet (so `999.1.1.1` and version strings like
`1.2.3.4000` are left alone), and `email` requires a dotted TLD. Anything site-specific belongs in `regex`.

| Option | Type | Default | Notes |
|---|---|---|---|
| `patterns` | []string | `[]` | Built-ins to enable: `email`, `ipv4`, `credit_card` |
| `regex` | []string | `[]` | Additional patterns (RE2 syntax). Every match is redacted |
| `mode` | string | `mask` | `mask` (replace), `hash` (salted SHA-256 digest), or `remove` (delete) |
| `replacement` | string | `[REDACTED]` | Replacement text in `mask` mode |
| `hash_salt` | string | `""` | Salt for `hash` mode. Agents sharing a salt produce equal digests for equal inputs, which is what makes correlation possible without retaining the value |
| `fields` | []string | `[]` | Restrict redaction to these field names. Empty = every field |
| `skip_fields` | []string | `[]` | Never redact these fields. Wins over `fields` |
| `skip_message` | bool | `false` | Leave the message untouched and redact only fields |

`hash` mode preserves correlation: the same value always yields the same digest, so you can still count
distinct users or join events without holding the identifier. Use a salt so the digests are not a
rainbow-table lookup of the original value.

```yaml
- type: redact
  patterns: [email, credit_card]
  regex:
    - 'token=[A-Za-z0-9]+'         # site-specific secrets
  mode: mask
  replacement: "[REDACTED]"
  skip_fields: [host]              # never scrub the routing metadata
```

#### `lookup` ✅ — enrich events from a local table (asset inventory, CMDB export) — **Apex**

Reads one field's value, finds the matching row in a table, and copies that row's columns onto the event.
Never drops events.

A hostname means something on the endpoint and nothing in a SIEM. Turning `host=web01` into
`owner=payments-team`, `env=prod`, `site=fra1` **at the source** lets downstream routing, alerting and
detection rules key on business facts instead of on a name someone has to look up by hand — and it costs the
aggregator nothing: no join, no enrichment pipeline, no per-event round trip.

The table is a CSV or JSON file you already have, or an inline `table:` for a handful of rows. Set exactly one
of `file:` or `table:`.

- **CSV** — first row is the header, **first column is the key**, remaining columns become fields named by the
  header.
- **JSON** — `{"web01": {"owner": "payments", "vlan": 42}, …}`. Numbers and booleans are rendered as text, so
  an inventory export works unmodified.

| Option | Type | Default | Notes |
|---|---|---|---|
| `key_field` | string | *(required)* | Field whose value is looked up. `host` / `hostname` / `source` fall back to the event's own host/source when no field of that name exists |
| `file` | string | `""` | Path to the table (CSV or JSON). Mutually exclusive with `table` |
| `format` | string | *(from extension)* | `csv` or `json`. Only needed when the filename doesn't say |
| `table` | map | `{}` | Inline table: `key → {field: value}`. Mutually exclusive with `file` |
| `fields` | []string | `[]` | Copy only these columns. Empty = every column in the row |
| `prefix` | string | `""` | Prepended to every added field name, e.g. `asset_` |
| `on_missing` | string | `skip` | `skip` (leave the event untouched) or `default` (apply `default:`) |
| `default` | map | `{}` | Values applied when the key is unknown. Required by `on_missing: default` |
| `overwrite` | bool | `false` | `false` keeps a field the event already carries; `true` replaces it |
| `reload` | duration | `0` | Re-read the file when it changes, checked at most this often. `0` = never |
| `case_insensitive` | bool | `false` | Match keys ignoring case (Windows hostnames arrive in mixed case) |

**A missing key is not an error.** An inventory always trails reality — a host built ten minutes ago isn't in
it yet — so by default the event passes through unchanged. Use `on_missing: default` when a downstream
consumer needs the field on every event.

**Reload is lazy and safe.** The file's timestamp is checked at most once per `reload` interval, and re-read
only when it actually changed; a table that is briefly unreadable (mid-rewrite, permissions churn) keeps the
copy already in memory, so enrichment degrades to *stale*, never to *absent*. A file that cannot be read at
**startup** is a configuration error instead — a lookup that enriches nothing should fail loudly, not quietly.

```yaml
- type: lookup
  key_field: host                  # `host` reads the event's own hostname
  file: /etc/logrok-agent/assets.csv
  format: csv                      # host,owner,env,site
  fields: [owner, env]             # ignore the other inventory columns
  prefix: asset_                   # -> asset_owner, asset_env
  reload: 5m                       # pick up inventory changes without a restart
  case_insensitive: true
  on_missing: default
  default: { owner: unassigned, env: unknown }
```

---

## `outputs`

One or **more** outputs. With a single output the agent behaves exactly as before. Listing several
outputs enables **multi-destination fan-out**: every event is delivered to *every* output, each with
its own independent disk spool, drain loop, and metrics — a dead or slow destination never blocks the
others, and each destination's backlog survives restarts on its own spool. Fan-out is a paid (Apex)
capability; single-destination forwarding stays free (Core).

### Fan-out (multiple outputs)

```yaml
outputs:
  - name: siem          # required to be unique per output (defaults to the type)
    # when: 'severity <= 3'   # optional routing: only matching events reach this destination
    type: syslog
    endpoint: "siem.example:6514"
  - name: lake
    type: hec
    endpoint: "https://splunk.example:8088"
    token: "REDACTED-HEC-TOKEN"
    buffer: {max_bytes: 1073741824, when_full: drop_oldest}   # per-output override
buffer:
  enabled: true
  dir: /var/lib/agent/spool     # per-destination subdirs: spool/siem, spool/lake
```

- **`name`** keys the destination's spool subdirectory and its metrics label. It defaults to the
  output's `type`; with more than one output every resolved name must be unique (two unnamed outputs
  of the same type are rejected with a naming hint). Names are restricted to letters, digits, `_`,
  `-`, `.` (no leading `-`/`.`) — they become directory names. Names must also stay unique after
  the metrics-key fold (lowercased; `-` and `.` become `_`): `a-b` and `a.b` would report into the
  same per-destination metrics series, so such pairs are rejected with the colliding key named.
- **Per-output `when:` routing** — an optional condition (the same expression grammar as the
  `expr`/`filter` processors: `severity <= 3`, `fields.app == "nginx"`, `message contains "auth"`,
  combined with `and`/`or`/`not`) restricts which events reach that destination. Absent = the
  destination receives every event. An event matching **no** destination is intentionally
  discarded (counted under the `routing_when` drop family) — **a `when:`-filtered event is gone; it
  is not spooled anywhere**, so keep a catch-all destination if you need one. A bad expression is
  rejected when the configuration is applied (a pushed config rolls back). Routing requires the
  same Apex entitlement as fan-out; a degraded agent clears `when:` conditions with a loud warning
  so it never silently drops events you believe are being collected.
- **Per-output `buffer:` override** — `max_bytes` and `when_full` may be set per output (a lossy
  best-effort sink and a must-not-drop SIEM feed legitimately want different policies). Anything not
  overridden inherits the global `buffer` settings. An explicit `max_bytes: 0` means unlimited.
- **Delivery semantics.** An input's checkpoint advances only when **every** destination holds the
  event durably (delivered or on its spool) — so no destination can silently miss events the agent
  has acknowledged to its source. Under `when_full: block`, a full destination back-pressures the
  whole pipeline (the slowest destination governs, by design — `block` means "never lose").
- **Spool layout & scaling.** With one output the spool lives directly in `buffer.dir` (unchanged
  from earlier releases). With several, each destination gets `buffer.dir/<name>`. The agent
  migrates automatically in both directions: going from one output to several moves an existing
  root-layout backlog into the *first* output's subdirectory; going back to one output moves that
  output's subdirectory back. A backlog is never silently stranded — subdirectories that match no
  configured output name are reported loudly at startup ("orphaned spool subdirectory"), and a
  migration that would overwrite existing spool files refuses and fails the config load instead.
- **Metrics.** Each destination exports its own series —
  `logrok_agent_output_events_out_total{output="siem"}`, `..._events_dropped_total`,
  `..._buffer_depth`, `..._spool_generation` — while the fleet-wide unlabeled totals remain the sums
  across destinations.
- Per-output routing/filtering (`when:` conditions) is not part of fan-out v1; every destination
  receives every event. Use per-destination processors downstream or a filter processor ahead of the
  outputs.

### `syslog` ✅ — RFC 5424 over TCP (TLS/mTLS) or UDP (diode mode)

The default output: forwards every event as **RFC 5424 syslog** — the vendor-neutral wire format any
aggregator, collector, or SIEM accepts (a logrok deployment's syslog-ng front door by default, but
equally any third-party receiver). TCP with TLS/mTLS is the normal production shape; UDP exists for
hardware data diodes and plain `udp()` receivers. Structured fields survive the trip as RFC 5424
structured data — or as a JSON message body with `encoding: json`.

| Option | Type | Default | Notes |
|---|---|---|---|
| `endpoint` | string | *(required)* | `host:port` of the aggregator (logrok syslog-ng by default) |
| `protocol` | string | `tcp` | `tcp` or `udp`. UDP sends **one datagram per message** (RFC 5426, no framing) — required for **hardware data diodes** (Waterfall/Owl pass UDP only) and plain `udp()` receivers. UDP is fire-and-forget: no delivery guarantee on the wire; pair with `sequence` for receiver-side gap detection |
| `framing` | string | `newline` | **TCP only.** `newline` (one message per LF — what syslog-ng's `network()` source and most receivers expect) or `octet-counting` (RFC 6587 `LEN SP MSG`; preserves embedded newlines but needs an octet-counting-aware receiver, e.g. syslog-ng's `syslog()` source). Setting it with `protocol: udp` is a config error |
| `sequence` | bool | `false` | add a `[seq@66371 session=".." n=".."]` SD element with a per-event monotonic counter. A receiver behind a one-way link detects **loss** (gaps in `n`) and **agent restarts** (`session` change). Counters are assigned at send time, so retried batches get fresh numbers — `n` detects gaps, not duplicates |
| `max_datagram_size` | int | `8192` | **UDP only.** Datagrams are truncated to this many bytes (an over-MTU/oversized send would otherwise fail forever and wedge the spool drain). A truncation counts `logrok_agent_udp_truncated_total` and logs a one-time WARN, so a too-small value is visible rather than silently cutting messages — raise it if your receiver accepts more |
| `encoding` | string | `text` | `text` (message verbatim, structured fields as a `[logrok@66371 ...]` SD element) or `json` (the RFC 5424 envelope stays, but MSG is **one JSON object** — `timestamp`, `host`, `source`, `severity`, `facility`, `message`, `fields` — and the fields SD element is dropped, no duplication). Use `json` for receivers that parse JSON natively (syslog-ng `json-parser()`, Splunk, Logstash, Loki). Bonus: JSON escapes control characters, so **multi-line messages survive newline framing** as a single frame — no `octet-counting` receiver needed. The `sequence` SD element still applies. ⚠️ **Keep `text` when forwarding into logrok**: its field extraction reads the `[logrok@66371]` SD element, which `json` drops — `json` would land fields (incl. `agent_group`) unparsed inside the message |
| `tls` | bool | `false` | enable TLS (**TCP only** — no DTLS; `protocol: udp` + `tls` is a config error) |
| `ca_file` | string | system roots | CA bundle to **verify the server** (RootCAs) |
| `cert_file` | string | `""` | client cert for **mTLS** (requires `key_file`) |
| `key_file` | string | `""` | client key for mTLS (requires `cert_file`) |
| `server_name` | string | from `endpoint` | verification name / SNI override |
| `cert_source` | string | `static` | `static` (use `cert_file`/`key_file`) or `enrolled` (present the control-plane-issued cert from `management.tls.mode: enrolled`; mutually exclusive with `cert_file`/`key_file`). The agent *presents* the cert; the receiver must verify it (`peer-verify`/`ca-file`) for enforcement |
| `insecure_skip_verify` | bool | `false` | **DEV ONLY** — disables server verification |
| `drain_pacing` | block | *(off)* | **Reconnect drain pacing** for this unacked transport. Omit → the spool drains at line rate (default). Set it when a high-volume plain-syslog feed can overrun a bounded receiver on reconnect: after an outage the durable spool would otherwise re-blast the whole backlog at once, and because plain syslog has **no delivery ack** the receiver's overflow drops are invisible to the agent (it can silently lose its own backlog). Pacing caps the drain rate and eases a recovered receiver back in. **Only for `syslog`/`snare`** (unacked); the acked outputs (`otlp`/`relay`, HEC-ack) are paced by their receiver's acknowledgment and ignore this. Sub-keys below |
| `drain_pacing.rate_eps` | int | *(required in block)* | Steady events/sec cap while **draining a backlog** (0/omitted = unlimited, i.e. the block does nothing). Steady-state forwarding at or below this rate is never throttled |
| `drain_pacing.slow_start` | bool | `true` | After each reconnect, start well below `rate_eps` and **double the rate every second** until the cap — so a just-recovered receiver is eased back in, not hit with the whole backlog at once. This is what breaks the receiver-restart-loop. Set `false` for a flat cap |
| `drain_pacing.ramp_start_eps` | int | `rate_eps / 16` (min 1) | Initial post-reconnect rate when `slow_start` is on; doubles each second up to `rate_eps`. Must be ≤ `rate_eps` |
| `drain_pacing.jitter` | bool | `true` | Add a small, per-agent random delay before resuming the drain after a reconnect, so a **fleet** of agents reconnecting to a recovered receiver at the same moment doesn't stampede it in lockstep |

Framing defaults to **newline-delimited** (interoperates with logrok's syslog-ng and most receivers; switch to
`octet-counting` only for a receiver that supports it). Structured `Fields` are emitted as RFC 5424
structured-data (`encoding: text`, default) or inside the JSON body (`encoding: json`). PARAM-VALUEs are
RFC 5424 §6.3.3-escaped (`"`, `\`, `]` backslashed), so field values containing brackets — common in
Windows Event Log text and stack traces — produce well-formed frames a strict parser accepts. Timestamps
carry a 6-digit microsecond `TIME-SECFRAC` (the RFC 5424 maximum), preserving sub-second ordering on the
wire. The wire TIMESTAMP is always **UTC** (the right default for cross-fleet correlation and downstream
storage); when the source's original timestamp carried a **non-zero UTC offset** (e.g. macOS `oslog`), that
offset is preserved losslessly as a `tz_offset="+02:00"` SD field, so "what was the source's local time" is
never lost — no wire-timestamp change, no config knob (UTC-source events omit it). When an event carries a
`pid` field (Windows Event Log, macOS `oslog`, journald), it is emitted in the RFC 5424 **PROCID** header
field — so a collector can index the process id as a first-class column rather than parse it out of structured
data (it also stays in SD). Events without a pid emit PROCID `-` (NILVALUE).
The event source (`file:<path>`, `journald:<unit>`, `oslog:<process>`, …) is rendered as the RFC 5424
**APP-NAME** header field, made grammar-valid on the way out: characters outside printable ASCII (incl.
spaces) become `_` and the value is capped at the RFC's 48-character maximum — a raw source like
`file:/var/log/my app.log` would otherwise shift every later header field on a spec-following receiver.
Lossless: whenever that sanitization changed the value, the full original source is added as a
`source="…"` SD param (`encoding: json` always carries it in the JSON body) — so APP-NAME is the
indexable shorthand, never the only copy. A user field already named `source` takes precedence.
The **HOSTNAME** header field gets the same treatment (it is peer-controllable via the `syslogin`/`relay_in`
receivers): bytes outside printable ASCII — including spaces, which would otherwise shift every later header
field — become `_`, the value is capped at the RFC's 255-byte maximum, and the full original rides as a
`host="…"` SD param when sanitization changed it (`encoding: json` carries it in the body). Structured-data
**field keys** (SD-NAME) — arbitrary via `parse_json`/`parse_kv`/`parse_csv` and `relay_in` — are likewise
made grammar-valid: out-of-charset bytes become `_`, keys cap at the RFC's 32-character maximum, and an empty
key becomes `_`. The **PRI** value is clamped defensively — severity into 0–7, facility into 0–23 — so a
peer-injected out-of-range value can never emit a malformed `<247>`/`<-1>` frame a strict receiver rejects.
Server verification is **on by default** when `tls: true`; `insecure_skip_verify` is the only way to disable it.

### `snare` ✅ — Snare-format (SnareCore) over syslog, for SIEM-legacy interop — **Apex**

Emits events as the legacy **Snare format** — the tab-delimited `MSWinEventLog` line that the Snare
Windows Agent and NXLog's `om_snare` produce, and that many SIEM ingest pipelines (rsyslog
`mmsnareparse`, Graylog, Splunk, QRadar) were built to parse. Reach for it when the receiving SIEM is
already configured for an NXLog/Snare feed and you want to swap the agent in without re-tooling the
SIEM side; otherwise the standard `syslog` output is the normal choice.

| Option | Type | Default | Notes |
|---|---|---|---|
| `endpoint` | string | *(required)* | `host:port` of the receiver (a SIEM / syslog collector that parses Snare — rsyslog `mmsnareparse`, Graylog, Splunk, QRadar) |
| `checksum` | string | `none` | trailing Snare field: `none` (the `0` placeholder — most parsers strip the last column regardless) or `md5` (the MD5, hex, of the record body — the classic Snare integrity column). MD5 here is a **format-compat checksum, not a security control** |
| `protocol` | string | `tcp` | `tcp` or `udp` — same transport as the `syslog` output (UDP = one datagram per message, for diodes / `udp()` receivers) |
| `framing` | string | `newline` | **TCP only**, same as `syslog`: `newline` (one line per LF — the normal Snare-over-syslog wire) or `octet-counting` (RFC 6587). A Snare record is single-line by construction (embedded tabs/newlines in a field are neutralized to spaces so they can't shift columns) |
| `max_datagram_size` | int | `8192` | **UDP only**, same as `syslog` |
| `tls` | bool | `false` | enable TLS (**TCP only**) — identical surface to the `syslog` output |
| `ca_file` | string | system roots | CA bundle to **verify the server** (RootCAs) |
| `cert_file` | string | `""` | client cert for **mTLS** (requires `key_file`) |
| `key_file` | string | `""` | client key for mTLS (requires `cert_file`) |
| `server_name` | string | from `endpoint` | verification name / SNI override |
| `cert_source` | string | `static` | `static` (use `cert_file`/`key_file`) or `enrolled` (present the control-plane-issued cert from `management.tls.mode: enrolled`; mutually exclusive with `cert_file`/`key_file`) |
| `insecure_skip_verify` | bool | `false` | **DEV ONLY** — disables server verification |

Emits the classic tab-delimited **"MSWinEventLog" Snare line** (Snare Windows Agent / NXLog `om_snare`
parity), so a SIEM configured for a legacy NXLog/Snare feed ingests the agent with no re-tooling. The line is
an RFC 3164 syslog header (`<PRI>Mmm dd HH:MM:SS HOST`) followed by the literal tag `MSWinEventLog` and 14
TAB-separated fields: `Criticality`, `LogName`, `Counter`, `DateTime`, `EventID`, `SourceName`, `UserName`,
`SIDType`, `EventLogType`, `ComputerName`, `CategoryString`, `DataString`, `ExpandedString`, `Checksum`.
The event maps in vendor-neutrally (works for non-Windows events with sensible fallbacks): `Criticality`
derives from syslog severity (0 = Clear … 4 = Critical); `LogName` ← `Fields["channel"]`; `Counter` is a
per-output monotonic Snare event counter; `EventID` ← `Fields["event_id"]` (else `0`); `SourceName` ←
`Fields["provider"]`; `UserName` ← `Fields["user"]`/`Fields["username"]`; `SIDType` ← `Fields["sid_type"]`;
`EventLogType` ← `Fields["event_type"]` (else severity-derived `Error`/`Warning`/`Information`);
`ComputerName` ← `Host`; `CategoryString` ← `Fields["category"]`; `DataString` ← `Fields["data"]`;
`ExpandedString` ← the message text. Absent string fields become the Snare-conventional `N/A` (or `0`/empty
where that is the convention). The transport (dial, TLS/mTLS, TCP/UDP framing, connection-liveness guards) is
the **same code path as the `syslog` output** — the Snare format is only a different line serializer, not a
second network stack.

### `otlp` ✅ — OpenTelemetry Protocol (gRPC or HTTP) — **Apex**

Forwards events as **OTLP logs** to an OpenTelemetry Collector or any backend with a native OTLP ingest
endpoint — the output to use when your downstream is OTel-native rather than syslog-speaking. Events
travel as structured `LogRecord`s with attributes, not flattened text lines, and the receiver's
per-batch acknowledgment feeds the same spool-and-retry reliability as every other output.

| Option | Type | Default | Notes |
|---|---|---|---|
| `endpoint` | string | *(required)* | grpc: `host:port` (e.g. `collector:4317`) · http: `http(s)://host:port[/v1/logs]` (e.g. `https://collector:4318`) |
| `protocol` | string | `grpc` | `grpc` (OTLP/gRPC — the gRPC wire protocol over HTTP/2, plaintext h2c or TLS) or `http` (OTLP/HTTP) |
| `tls` | bool | `false` | **grpc only** — TLS on/off (http carries it in the endpoint URL scheme instead) |
| `insecure_skip_verify` | bool | `false` | **DEV ONLY** — disables server verification |
| `ca_file` | string | system roots | CA bundle to **verify the server** (RootCAs) |
| `cert_file` | string | `""` | client cert for **mTLS** (requires `key_file`) |
| `key_file` | string | `""` | client key for mTLS (requires `cert_file`) |
| `server_name` | string | from `endpoint` | verification name / SNI override |
| `cert_source` | string | `static` | `static` (use `cert_file`/`key_file`) or `enrolled` (present the control-plane-issued cert from `management.tls.mode: enrolled`; mutually exclusive with `cert_file`/`key_file`) |
| `headers` | map[string]string | `{}` | extra gRPC metadata / HTTP headers, e.g. an `Authorization` bearer token for a SaaS OTLP receiver. The reserved keys `content-type`/`content-encoding` (case-insensitive) are **rejected at config-build time** — the transport sets those itself |
| `compression` | string | `gzip` | `gzip` (stdlib `compress/gzip`) or `none` |
| `encoding` | string | `protobuf` | **http only.** `protobuf` (binary OTLP) or `json` (proto3-JSON body). Setting `encoding` with `protocol: grpc` is a config error — gRPC is always protobuf on the wire |
| `timeout` | duration | `10s` | per-request export deadline |
| `on_reject` | string | `drop` | what happens to a batch the receiver **permanently** rejects (schema error, auth failure — a retry can never fix it): `drop` (log and discard) or `dead_letter` (quarantine to disk) |
| `dead_letter_dir` | string | `""` | required when `on_reject: dead_letter` — directory rejected batches are written to as proto3-JSON, one file per batch. **Capped at 100 MiB total**; once the cap is hit, further rejected batches are dropped (loudly logged) rather than growing the directory unbounded |

Forwards logs to any OTLP-compliant receiver — an OpenTelemetry Collector, or any backend with a native OTLP
ingest endpoint — over **gRPC** (`connectrpc.com/connect`, the gRPC wire protocol on top of `net/http`; no
`grpc-go` in the dependency graph) or **OTLP/HTTP**. Both protocols carry a structured `LogRecord` per event
(severity, timestamp + observed-timestamp, resource attributes for host/source, `syslog.procid`/`syslog.msgid`/
`syslog.facility` attributes) rather than a flattened text line — the natural fit for OTel-native platforms.
Batches are grouped into `resourceLogs` by `(host, source)`.

**Delivery semantics.** Unlike plain syslog/TCP, `Write` gets a **per-batch application-level acknowledgment**
from the receiver: gRPC status codes / HTTP status + the OTLP `partialSuccess` field tell the agent whether the
batch was actually accepted, not just TCP-received. A **transient** failure (network error, HTTP `429`/`502`/
`503`/`504`, most gRPC codes) returns to the pipeline, which spools it to disk and retries — the same
store-and-forward guarantee as every other output. A **permanent** failure (gRPC `InvalidArgument`,
`NotFound`, `AlreadyExists`, `PermissionDenied`, `Unauthenticated`, `FailedPrecondition`, `Unimplemented` — the
codes that can only mean "the server made a final decision, retrying changes nothing") is never retried:
`on_reject` decides whether the batch is dropped or written to `dead_letter_dir` for offline inspection (the
100 MiB cap is a reliability bound, not automated replay — dead-letter is diagnostic quarantine). A partial
success (some records rejected, others accepted) is logged as a warning, per the OTLP spec, and not retried.
Redirects are never followed. One bound applies to any acknowledged transport: an acknowledgment means the
receiver *accepted* the batch, so the agent's delivery guarantee ends at the receiver's front door — what
happens downstream of an ack is bounded by the receiver's own ack-to-durability depth. A receiver that
acknowledges into an in-memory queue can still lose acked batches if it crashes before flushing them onward;
where that matters, enable durable buffering on the receiver's onward path.

**Plaintext gRPC (h2c)** works out of the box against a stock `otel/opentelemetry-collector` OTLP receiver on
its default `:4317` — set `protocol: grpc` with `tls: false` for a LAN/dev collector, add `tls: true` (+
`ca_file`/mTLS) once TLS terminates in front of it. `encoding` only applies to `protocol: http`; gRPC is always
binary protobuf on the wire.

### `hec` ✅ *(verified end-to-end against a real Splunk instance in both commit modes, including sustained-outage recovery with zero event loss)* — Splunk HTTP Event Collector — **Apex**

Forwards events to **Splunk's HTTP Event Collector (HEC)** — the standard Splunk ingest path, and the
sink to reach for when the destination is Splunk rather than a syslog-speaking aggregator. Two commit
modes: the **default (HTTP-response) mode** commits a batch on a 2xx response — this means Splunk
*received* the batch into its ingestion pipeline, not that it has been *indexed* yet. The **opt-in
indexer-acknowledgment mode** (`use_ack: true`) instead commits only once Splunk confirms the batch was
actually **indexed**, at the cost of one or more acknowledgment poll round trips per batch and a double
opt-in requirement (the flag here *and* acknowledgment enabled on the HEC token).

| Option | Type | Default | Notes |
|---|---|---|---|
| `endpoint` | string | *(required)* | `http(s)://host:port[/services/collector/event]` — the path is implied when omitted; the explicit canonical `/services/collector/event` path is accepted (including behind a reverse-proxy prefix). Any other `/services/collector/*` path is a config error — the raw endpoint is not supported (the agent sends event-format JSON only) |
| `token` | string | *(required)* | the HEC token; sent as `Authorization: Splunk <token>` |
| `index` | string | `""` | target index; omit to use the token's default index |
| `sourcetype` | string | `""` | HEC sourcetype; omit to use Splunk's default for the token |
| `source` | string | `""` | HEC source; omit to use the event's own `Source` (e.g. `file:/path`, `windows-eventlog:Channel`) |
| `event_format` | string | `message` | `message` (event text is the message body; structured fields ride as HEC *indexed fields*) or `json` (the whole structured event — message, severity, facility, host, source, fields — as the body) |
| `gzip` | bool | `false` | send POSTs with `Content-Encoding: gzip` (WAN-friendly) |
| `timeout` | duration | `10s` | per-request HTTP timeout |
| `batch_max_bytes` | int | `524288` (512 KiB) | batching granularity / memory bound. A single event larger than this can never be sent and is terminally rejected (dead-lettered + counted dropped) rather than retried forever |
| `use_ack` | bool | `false` | opt-in indexer acknowledgment — commit only once Splunk reports the batch **indexed** (the HEC token must also have acknowledgment enabled) |
| `ack_poll_every` | duration | `2s` | how often the agent polls Splunk for ack status in `use_ack` mode. Must not exceed `ack_timeout` (rejected at config-load) |
| `ack_timeout` | duration | `2m` | how long the agent waits for `ack=true` before treating the batch as failed (spool + retry) |
| `ca_file` | string | system roots | CA bundle to verify the server cert (`https://` endpoints only) |
| `insecure_skip_verify` | bool | `false` | **DEV ONLY** — disables server certificate verification (`https://` endpoints only) |
| `dead_letter_dir` | string | `""` | directory terminally-rejected (malformed) events are written to as newline-delimited JSON (NDJSON, one event object per line); omit to drop them (still counted). Capped at 100 MiB total — once full, further rejections are dropped and logged rather than growing the directory unbounded |

**Indexer acknowledgment.** HTTP-response mode and indexer-ack mode answer different questions: a 2xx
means Splunk's HEC endpoint *received* the batch, while `use_ack: true` waits for Splunk's ack poller to
confirm the batch was *indexed* — the stronger guarantee, but only available when both this flag is set
**and** the HEC token has acknowledgment enabled on the Splunk side (double opt-in; with `use_ack: false`
the output behaves identically against any standard token). While a batch is unacknowledged, `Write`
blocks the pipeline batch — if it never reaches `ack=true` within `ack_timeout`, the batch is treated as
failed and spooled for retry, which means the retried batch can be **re-delivered and re-indexed as a
duplicate** (at-least-once, not exactly-once — the same honesty as every other output's reliability
story). On a slow indexer, prefer **raising `ack_timeout`** over leaving it low: a too-tight timeout
converts ordinary indexing latency into repeated duplicate resends rather than actually protecting
anything. `ack_poll_every` must not exceed `ack_timeout` — the agent refuses to start with a poll interval
that can never fire before the deadline.

**Error handling.** Only a 400 response carrying one of HEC's proven-malformed-payload codes (no data,
invalid data format, missing/blank event field, indexed-fields error) is treated as unrecoverable: those
events are dead-lettered (or dropped, if `dead_letter_dir` is unset) and counted in `events_dropped` —
retrying data HEC has already proven malformed helps nobody. Every other failure — authentication/token
errors, an incorrect index, channel/acknowledgment misconfiguration, HEC reporting itself busy, any
unrecognized response code, and ordinary network/timeout failures — is treated as **retryable**: the batch
stays in the disk spool and is retried, and events are never dropped for a condition the operator can fix
(e.g. a bad token) or that is merely transient.

### `loki` ✅ *(verified end-to-end against a real Grafana Loki instance — events query-verified after delivery, and after a sustained receiver outage with zero event loss)* — Grafana Loki — **Apex**

Pushes events to Loki's ingest API (`POST /loki/api/v1/push`).

| Option | Type | Default | Notes |
|---|---|---|---|
| `endpoint` | string | *(required)* | `http(s)://host:3100` (the push path is implied) or the explicit `…/loki/api/v1/push` when a reverse proxy adds a prefix |
| `tenant` | string | `""` | Sent as `X-Scope-OrgID`. Required by a multi-tenant Loki |
| `labels` | map | `{job: luna}` | Static stream labels. **This is the safe place to put labels** |
| `label_fields` | []string | `[]` | Event fields promoted to labels. `host` and `source` fall back to the event's own values. **Read the cardinality warning below** |
| `structured_metadata` | bool | `false` | Send the remaining fields as Loki structured metadata — queryable **without** being indexed |
| `line_format` | string | `message` | `message` (the log line) or `json` (the whole event as a JSON line) |
| `gzip` | bool | `false` | Compress the request body |
| `timeout` | duration | `10s` | Per-request timeout |
| `batch_max_bytes` | int | `1048576` | Split larger batches into several pushes; stays under the usual distributor receive limit |
| `bearer_token` | string | `""` | `Authorization: Bearer …`. Mutually exclusive with basic auth |
| `basic_auth_user` / `basic_auth_password` | string | `""` | HTTP basic auth (Grafana Cloud uses the user for the tenant/instance ID) |
| `ca_file` | string | `""` | CA bundle for `https://` endpoints |
| `insecure_skip_verify` | bool | `false` | Skip certificate verification (test only) |
| `dead_letter_dir` | string | `""` | Where payloads Loki permanently rejected are written instead of being dropped silently |

**Label cardinality is the one way to hurt a Loki cluster from the agent.** Loki builds an index entry per
distinct label set, so promoting a per-event value — a request ID, a user ID, a session — to a label
multiplies streams without bound and can take ingesters down. Keep `labels` static, put per-event detail in
`structured_metadata` (queryable, not indexed), and use `label_fields` only for genuinely low-cardinality
dimensions like environment or role. The agent warns in its own log the first time one batch produces an
unusually large number of streams, so the mistake surfaces on day one instead of in an ingester post-mortem.

**Ordering.** Loki accepts out-of-order entries by default (since 2.4), so the agent neither sorts nor holds
events back. An entry older than the cluster's accepted window is rejected as a validation error like any
other malformed payload.

**Error handling.** A `400` is documented as a validation failure Loki will never accept: that payload is
dead-lettered (or dropped, if `dead_letter_dir` is unset) and counted in `events_dropped`, because retrying it
forever would wedge the spool behind data that can never land. Everything else — `429` (the ingestion rate
limit), any `5xx`, authentication failures, and ordinary network/timeout errors — is **retryable**: the batch
stays in the disk spool and is retried, so a rate limit or a fixable credential never costs events.

```yaml
- type: loki
  endpoint: "http://loki.example:3100"
  tenant: acme
  labels: { job: luna, env: prod }
  label_fields: [host]          # low cardinality only
  structured_metadata: true     # everything else rides here
  gzip: true
```

### `kafka` ✅ — Apache Kafka / Kafka-compatible brokers (Redpanda, …) — **Apex**

Produces events to a Kafka topic — the sink to reach for when the destination is a Kafka-based
pipeline or data lake rather than a syslog receiver. Acked like `otlp`/`hec`: the producer's
per-record delivery report commits the batch, transient broker failures (down broker, leader
election, throttling) spool-and-retry with at-least-once delivery, and idempotent produce is
always on (the client's internal retries cannot duplicate records on the broker). Authentication
and authorization failures also spool-and-retry — fixed credentials or ACLs deliver the backlog
rather than finding it discarded. Only payload-inherent rejections (a record larger than the
cluster accepts, a record the broker's validation refuses) are terminally disposed via
`on_reject`.

| Option | Type | Default | Notes |
|---|---|---|---|
| `brokers` | list | *(required)* | seed broker list, `host:port` each (not URLs) |
| `topic` | string | *(required)* | target topic; the agent never auto-creates topics (a broker policy, not an agent right) |
| `encoding` | string | `json` | `json` (the whole structured event — timestamp, host, source, severity, facility, message, fields — as one JSON object) or `message` (the raw message text only) |
| `key_field` | string | `host` | event field used as the record key (partition affinity — the default keeps per-host ordering). `host`/`source` read the event's own attributes; any other name reads a structured field. Explicit `key_field: ""` disables keying (round-robin) |
| `headers` | map | `{}` | static record headers attached to every record |
| `compression` | string | `snappy` | `snappy` \| `gzip` \| `lz4` \| `zstd` \| `none` |
| `tls` | bool | `false` | TLS to the brokers; sub-keys below apply when set |
| `ca_file` / `cert_file` / `key_file` / `server_name` / `insecure_skip_verify` | | | the standard output TLS block (mTLS via cert+key) |
| `cert_source` | string | `static` | `enrolled` presents the control-plane-issued client certificate (requires `management.tls.mode: enrolled`) |
| `sasl` | block | *(off)* | SASL authentication: `sasl.mechanism` (`plain` \| `scram-sha-256` \| `scram-sha-512`) + `sasl.username` + `sasl.password`, all required together |
| `timeout` | duration | `10s` | per-record delivery bound; past it the batch fails and is spooled for retry |
| `batch_max_bytes` | int | `1048576` (1 MiB) | producer batch bound. A single record larger than the cluster's message cap can never be sent and is terminally rejected rather than retried forever |
| `on_reject` | string | `drop` | disposal for terminally rejected records: `drop` (counted + logged) or `dead_letter` |
| `dead_letter_dir` | string | `""` | required with `on_reject: dead_letter`; rejected records are quarantined as replayable NDJSON (`{"topic","key","value"}` per line), bounded at 100 MiB |

```yaml
outputs:
  - type: kafka
    brokers: ["broker-1:9092", "broker-2:9092"]
    topic: logs
    sasl: {mechanism: scram-sha-512, username: agent, password: "REDACTED"}
    tls: true
    ca_file: /etc/logrok-agent/kafka-ca.pem
```

### `s3` ✅ — Amazon S3 / S3-compatible object stores (MinIO, Ceph, R2, …) — **Apex**

Writes event batches as objects to an S3 bucket — the sink for data-lake and archive patterns
(Athena, Snowflake, security-data-lake tooling all ingest the produced layout directly). Each
batch becomes one time-partitioned NDJSON object (gzipped by default), uploaded with a single
signed `PutObject`; the store's 200 response commits the batch. Transient failures (network, 5xx,
throttling) spool-and-retry with at-least-once delivery, and authentication, permission, and
missing-bucket failures also spool-and-retry — fixed credentials or a created bucket deliver the
backlog rather than finding it discarded. The agent never creates buckets (a store policy, not an
agent right). Only a payload-inherent rejection (an object larger than the store accepts) is
terminally disposed via `on_reject`. Works against any S3-API endpoint: leave `endpoint` empty
for AWS, or point it at a compatible store.

| Option | Type | Default | Notes |
|---|---|---|---|
| `bucket` | string | *(required)* | target bucket; must already exist |
| `region` | string | *(required for AWS)* | SigV4 signing region; defaults to `us-east-1` when `endpoint` is set (compatible stores accept any region string) |
| `endpoint` | string | *(AWS)* | S3-compatible store URL (e.g. `https://minio.example:9000`); empty targets `https://<bucket>.s3.<region>.amazonaws.com` |
| `force_path_style` | bool | auto | bucket in the path (`/bucket/key`) instead of the virtual host. Defaults to `true` when `endpoint` is set, `false` for AWS |
| `key_prefix` | string | `""` | prepended to every object key (e.g. `logs/`) |
| `partition_style` | string | `hive` | `hive` (`year=2026/month=08/day=18/hour=03/…` — Athena/Glue partition-friendly) or `path` (`2026/08/18/03/…`) |
| `encoding` | string | `json` | `json` (one JSON object per line — same object shape as the `kafka` output's `json` encoding, so one consumer schema covers both) or `message` (raw message lines) |
| `compression` | string | `gzip` | `gzip` (objects get `.ndjson.gz` and `Content-Encoding: gzip`) or `none` |
| `storage_class` | string | *(bucket default)* | optional `x-amz-storage-class` passthrough (e.g. `STANDARD_IA`) |
| `access_key_id` / `secret_access_key` / `session_token` | string | *(env)* | credentials; any field left empty falls back to `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_SESSION_TOKEN` |
| `ca_file` | string | `""` | CA bundle for a private endpoint's TLS |
| `insecure_skip_verify` | bool | `false` | skip TLS verification (testing only) |
| `timeout` | duration | `30s` | per-upload bound; past it the batch fails and is spooled for retry |
| `on_reject` | string | `drop` | disposal for terminally rejected objects: `drop` (counted + logged) or `dead_letter` |
| `dead_letter_dir` | string | `""` | required with `on_reject: dead_letter`; the rejected object is quarantined verbatim (already-replayable NDJSON), bounded at 100 MiB |

**Security-Lake feed (`format: parquet-ocsf`).** The second format turns the sink into an
[AWS Security Lake custom source](https://docs.aws.amazon.com/security-lake/latest/userguide/custom-sources.html)
feed: events render as **OCSF `base_event` (1.3.0)** rows and upload as **zstd-compressed Parquet**
objects under the mandatory `region=/accountId=/eventDay=` partition layout, time-sorted, one
eventDay per object. Because Security Lake wants objects accumulated over at least five minutes,
rows first land in a local fsync'd staging file — the batch is acknowledged once staged durably —
and an uploader flushes them on `flush_interval` (or earlier at `flush_max_bytes`). Staged rows
survive restarts and failed uploads (retried on the next interval; duplication possible after a
crash, never loss). Works against MinIO and other S3-compatible stores for testing.

| Option (parquet-ocsf mode) | Type | Default | Notes |
|---|---|---|---|
| `format` | string | `ndjson` | `ndjson` (one batch, one object — above) or `parquet-ocsf` (the Security-Lake feed) |
| `ocsf_class` | string | `base_event` | the OCSF class rendered; `base_event` is the supported class today |
| `account_id` | string | *(required)* | the Security-Lake `accountId` partition value (an AWS account id, or an `external`-style string for non-AWS sources) |
| `staging_dir` | string | *(required)* | local directory for the fsync'd staging file between flushes |
| `flush_interval` | duration | `5m` | upload cadence (Security Lake's documented delivery floor) |
| `flush_max_bytes` | int | `268435456` (256 MB) | staged-bytes early-flush trigger (the vendor's row-group ceiling) |

With `format: parquet-ocsf`, `partition_style` is not configurable (the Security-Lake layout is
mandatory) and `key_prefix` should carry the source prefix Security Lake assigns (e.g. `ext/<source>/`).

```yaml
outputs:
  - type: s3
    bucket: security-lake-raw
    region: eu-west-1
    key_prefix: logrok/
    # endpoint: https://minio.example:9000   # any S3-compatible store

  # AWS Security Lake custom-source feed:
  # - type: s3
  #   bucket: aws-security-data-lake-...
  #   region: eu-west-1
  #   format: parquet-ocsf
  #   account_id: "123456789012"
  #   key_prefix: ext/logrok/
  #   staging_dir: /var/lib/logrok-agent/s3-staging
  #   # flush_interval: 5m
```

### `sentinel` ✅ — Microsoft Sentinel / Azure Monitor Logs (Logs Ingestion API) — **Apex**

Sends events to a Log Analytics workspace — and thereby Microsoft Sentinel — via the modern **Logs
Ingestion API** with a Data Collection Rule (DCR). Batches are posted as JSON arrays (gzipped by
default, split automatically under the API's per-call size bound); the service's success response
commits the batch. Transient failures (network, 5xx, throttling) spool-and-retry with at-least-once
delivery, and authentication failures — both at the ingestion endpoint and at the token endpoint —
also spool-and-retry, so credentials fixed later deliver the backlog rather than finding it
discarded. Only a payload rejection (the rows don't match the DCR's stream declaration) is
terminally disposed via `on_reject`. Rows carry a fixed schema (`TimeGenerated`, `Computer`,
`Source`, `SeverityNumber`, `Facility`, `Message`, `AdditionalContext`); the DCR's `transformKql`
reshapes them onto your table, so no per-table configuration lives in the agent. Authentication is
a Microsoft Entra app registration (client-credentials flow) with the **Monitoring Metrics
Publisher** role on the DCR.

| Option | Type | Default | Notes |
|---|---|---|---|
| `endpoint` | string | *(required)* | the DCR's logs-ingestion endpoint URL (DCRs created as `kind: Direct`), or a Data Collection Endpoint for private-link setups |
| `dcr_id` | string | *(required)* | the DCR **immutable id** (`dcr-…`, from the DCR's overview page) |
| `stream` | string | *(required)* | the DCR stream name the rows target (e.g. `Custom-AgentLogs_CL`) |
| `tenant_id` / `client_id` / `client_secret` | string | *(env)* | Entra app-registration credentials; any field left empty falls back to `AZURE_TENANT_ID` / `AZURE_CLIENT_ID` / `AZURE_CLIENT_SECRET` |
| `scope` | string | `https://monitor.azure.com/.default` | token audience; override for sovereign clouds (`https://monitor.azure.cn/.default`, `https://monitor.azure.us/.default`) |
| `compression` | string | `gzip` | `gzip` or `none` |
| `timeout` | duration | `30s` | per-request bound (token and ingestion calls); past it the batch fails and is spooled for retry |
| `batch_max_bytes` | int | `1048576` (1 MB) | client-side split threshold matching the API's per-call bound (uncompressed) |
| `ca_file` | string | `""` | CA bundle for a private endpoint's TLS |
| `insecure_skip_verify` | bool | `false` | skip TLS verification (testing only) |
| `on_reject` | string | `drop` | disposal for terminally rejected rows: `drop` (counted + logged) or `dead_letter` |
| `dead_letter_dir` | string | `""` | required with `on_reject: dead_letter`; the rejected JSON array is quarantined verbatim (replayable against a fixed DCR with any HTTP client), bounded at 100 MiB |

```yaml
outputs:
  - type: sentinel
    endpoint: https://my-dcr-abcd.westeurope-1.ingest.monitor.azure.com
    dcr_id: dcr-000a00a000a00000a000000aa000a0aa
    stream: Custom-AgentLogs_CL
    # credentials fall back to AZURE_TENANT_ID / AZURE_CLIENT_ID / AZURE_CLIENT_SECRET
```

### `xsiam` ✅ — Palo Alto Cortex XSIAM (HTTP log collector) — **Apex**

Sends events to a Cortex XSIAM tenant via its HTTP log collector — batches post as
newline-delimited records (gzipped by default, split automatically under the vendor-recommended
1 MB request size), authenticated with the collector instance's API key. The success response
commits each batch. Transient failures, throttling, wrong-endpoint and authentication failures all
spool-and-retry — a rotated key or corrected URL delivers the backlog rather than finding it
discarded. Only a single record the collector's per-record size cap can never accept is terminally
disposed via `on_reject`. Note the collector reports a **log-format mismatch as a 500**, which the
agent deliberately treats as retryable: a persistently wedged xsiam spool usually means the
collector instance's configured format doesn't match `encoding` — fix the instance (or `encoding`)
and the backlog delivers.

| Option | Type | Default | Notes |
|---|---|---|---|
| `endpoint` | string | *(required)* | the tenant collector URL (`https://api-<tenant>/logs/v1/event`) |
| `api_key` | string | *(env)* | the collector instance's key; falls back to `XSIAM_API_KEY` |
| `encoding` | string | `json` | `json` (one JSON object per line — the same shape as the `kafka`/`s3` json encodings; use with a JSON-format collector instance) or `message` (raw log lines, for a Raw-format instance) |
| `compression` | string | `gzip` | `gzip` or `none` — **must match the collector instance's setting** (chosen when the instance is created) |
| `timeout` | duration | `30s` | per-request bound; past it the batch fails and is spooled for retry |
| `batch_max_bytes` | int | `1048576` (1 MB) | client-side split threshold (the vendor recommendation; hard request cap is 10 MB) |
| `ca_file` | string | `""` | CA bundle for TLS to the collector |
| `insecure_skip_verify` | bool | `false` | skip TLS verification (testing only) |
| `on_reject` | string | `drop` | disposal for terminally rejected records: `drop` (counted + logged) or `dead_letter` |
| `dead_letter_dir` | string | `""` | required with `on_reject: dead_letter`; rejected records are quarantined as replayable NDJSON, bounded at 100 MiB |

```yaml
outputs:
  - type: xsiam
    endpoint: https://api-mytenant.xdr.paloaltonetworks.com/logs/v1/event
    # api_key falls back to XSIAM_API_KEY
```

### `relay` ✅ — ack'd reliable transport (agent→agent)

Sends events to another agent — a [`relay_in`](#relay_in---cross-platform-the-ackd-transport-receiver)
gateway/concentrator — over the agent's own acknowledged protocol. Use it for the hops where plain TCP
syslog isn't safe enough (an unreliable WAN, an edge→hub leg): a TCP write can "succeed" while the peer
loses the last events on a broken connection, whereas here a write succeeds only after the receiver
acknowledges the batch. The final hop to any third-party aggregator stays standard RFC 5424.

| Option | Type | Default | Notes |
|---|---|---|---|
| `endpoint` | string | *(required)* | `host:port` of a [`relay_in`](#relay_in---cross-platform-the-ackd-transport-receiver) agent (gateway/concentrator) |
| `tls` | bool | `false` | enable TLS — same surface as the `syslog` output: `ca_file`, `cert_file`+`key_file` (mTLS), `server_name`, `insecure_skip_verify` (dev only) |
| `cert_source` | string | `static` | `static` (use `cert_file`/`key_file`) or `enrolled` (present the control-plane-issued cert from `management.tls.mode: enrolled`; mutually exclusive with `cert_file`/`key_file`). The agent *presents* the cert; the receiver must verify it (`peer-verify`/`ca-file`) for enforcement |
| `compress` | bool | `false` | gzip DATA frame bodies (stdlib `compress/gzip`) |
| `ack_timeout` | duration | `10s` | per-attempt wait for the receiver's acknowledgment |

The agent's answer to syslog-ng PE's proprietary **ALTP**: length-prefixed
`DATA`/`ACK`/`NACK`/`HEARTBEAT` frames over TCP/TLS, monotonic per-session sequence numbers. `Write` succeeds
**only after the receiver acknowledges the batch** — the application-level ack a plain TCP write can't give —
so the disk spool + retry now covers the *peer*, not just the wire: a TLS-rejected client cert, a receiver
crash mid-batch, or a connection drop before the ACK all land back in the spool. On a dropped connection the
output reconnects and **replays the same sequence number**, which the receiver deduplicates — a lost ACK never
duplicates events. A receiver **NACK** fails the write immediately (re-spool, no blind retry). Does **not**
replace plain syslog: the receiver must be a `relay_in` agent; the final hop to any third-party aggregator
stays standard RFC 5424.

---

## Minimal examples

**Linux gateway — receive syslog, forward over mTLS:**
```yaml
service: { mode: standalone, log_level: info }
buffer:  { enabled: true, dir: /var/lib/logrok-agent/spool, max_bytes: 1073741824 }
inputs:
  - { type: syslog_in, listen: "0.0.0.0:5514", protocols: [tcp, udp] }
outputs:
  - type: syslog
    endpoint: "aggregator:6514"
    tls: true
    ca_file: /etc/logrok-agent/ca.pem
    cert_file: /etc/logrok-agent/agent.pem
    key_file: /etc/logrok-agent/agent.key
```

**Diode mode — one-way link (data diode / unidirectional gateway) out of an OT zone:**
```yaml
# Send-only profile: UDP (diodes pass UDP only), sequence SD for receiver-side
# gap detection, disk spool so a NIC/link outage loses nothing locally, and no
# management endpoint (nothing can reach back through a one-way link).
service: { mode: standalone, log_level: info }
buffer:  { enabled: true, dir: /var/lib/logrok-agent/spool, max_bytes: 1073741824, when_full: block }
inputs:
  - { type: syslog_in, listen: "0.0.0.0:5514", protocols: [tcp, udp] }
  - { type: filetail, path: "/var/log/scada/*.log" }
outputs:
  - type: syslog
    endpoint: "diode-ingress:514"
    protocol: udp
    sequence: true
```

**Windows endpoint — Event Log → logrok, drop debug/info:**
```yaml
service: { mode: service, log_level: info }
buffer:  { enabled: true, dir: 'C:\ProgramData\logrok-universal-agent\spool' }
inputs:
  - { type: windows_eventlog, channels: [Application, System, Security] }
processors:
  - { type: filter, max_severity: 5 }
outputs:
  - { type: syslog, endpoint: "aggregator:6514", tls: true,
      ca_file: 'C:\ProgramData\logrok-universal-agent\ca.pem' }
```
