# LUnA — Configuration reference

The agent reads one YAML file (`-config <path>`, or `LOGROK_UNIVERSAL_AGENT_CONFIG`, default `agent.yaml`).
A working template is [`configs/agent.example.yaml`](../configs/agent.example.yaml). The same schema applies on
every platform.

**Shape.** Top-level sections are `service`, `management`, `buffer`, `licensing`, `limits`, `inputs`, `processors`, `outputs`. Each
`inputs` / `processors` / `outputs` entry has a `type`, an optional `name`, and then **that module's own
options inline** — adding a module type never changes the schema.

**Values are literal.** The agent does not expand environment variables: `"${TOKEN}"` is sent as those
eight characters. Write secrets into the file (it is read only by the agent's service account), or have your
deployment tooling render the file. The examples below use `REPLACE_WITH_…` for values you must fill in.

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

> **Compatibility.** A configuration written for the previous minor version keeps working unchanged
> (see the [support policy](SUPPORT-POLICY.md#configuration-compatibility)): keys are deprecated with a
> warning for at least one minor version and removed only at a major version. Validate a file against
> the binary you run with `logrok-universal-agent -check-config -config agent.yaml` — it prints a
> one-line verdict and names any key, module type or module option the binary does not know — it
> constructs every module exactly as the agent would at start, without opening a socket or a file (unlike
> the agent's own start, which tolerates an unknown key rather than refuse to run).

## `service`

| Key | Type | Default | Status | Notes |
|---|---|---|---|---|
| `mode` | string | `service` | ✅ | informational: defaults to `service`; the agent does not act on it (whether it runs as a Windows Service or in the foreground depends on how it is started) |
| `log_level` | string | `info` | ✅ | `debug` \| `info` \| `warn` \| `error` (structured `slog` JSON) |
| `metrics_listen` | string | `""` (off) | ✅ | e.g. `127.0.0.1:9090` → serves Prometheus text at `/metrics` **and the live-sampling endpoint `/tap`** (see [Watching live events](USER-GUIDE.md#watching-live-events--tap)). **Bind localhost** unless scraped remotely — `/tap` is refused outright on a non-loopback bind, because it exposes event content. |
| `batch_size` | int | `0` (= 256) | ✅ | events handed to each destination per write (and per spool-drain read), 1–16384. Each destination has one write in flight at a time, so against a receiver with a noticeable round trip a larger batch raises throughput: measured into an OpenTelemetry collector over loopback, one agent went from about 66,500 to 79,000 events/s at 2048. A batch is held in memory whole; outputs whose receivers cap the request size split a larger batch (`batch_max_bytes` on `otlp`, `hec`, `elasticsearch`, `loki`, `sentinel`, `xsiam`; `beats` splits windows over 64 MiB) |
| `batch_timeout` | duration | `0` (= `1s`) | ✅ | send a partly filled batch after this long; lower it for lower latency on quiet sources |
| `workers` | int | `0` (= 1) | ✅ | how many events the processors work on at once. `1` (the default) processes and sends as in earlier releases. `2`–`64` runs the **leading** processors that are safe to run concurrently — `add_fields`, `filter`, `expr`, `parse_json`, `parse_csv`, `parse_kv`, `parse_xml`, `redact`, `trim_fields`, `lookup`, `file_hash`, `noise_fingerprint`, and `sample` with a `key` — on that many workers, runs every other processor in arrival order on one worker (`dedup`, `throttle`, `quota`, `sample` without a `key`, `adaptive_sample`, `aggregate`, and everything after the first of them). Events reach each destination in the same order and in the same batches as with `1`, and an input's saved position still advances only after its events are delivered or durably spooled. The first processor that must run in order ends the parallel part, so **list the parallel-safe processors first**; the agent logs the split at start-up and warns when nothing can run in parallel. Worth raising only when the processors are what holds the agent back (check CPU); **4–8 is the practical range; more workers than CPUs gives no gain.** A value above the CPUs the agent may use (or above `limits.max_procs`) is applied as written and logs a warning. Up to about 2 × `workers` more batches are held in memory. When the receiver is far away, raise `batch_size` first |
| `shutdown_grace` | duration | `0` (= `15s`) | ✅ | how long the agent, when stopping, may spend delivering the last batch it holds before it gives up. Range `1s`–`10m`; `0` or absent = `15s`. When it expires the undelivered batch is not marked as sent: a replay-capable input (file, journal, event log) reads it again after the restart, but an input that cannot be replayed (network receivers such as `syslog_in`) loses that last unshipped batch. Raise it for a slow or distant receiver; the service manager's stop timeout must be at least as long |
| `log_file` | string | `""` (stdout) | ✅ | append the structured log to this file instead of stdout, rotated once when it passes 50 MiB (`<file>.1` replaces the previous one). Go crash reports (an unrecovered panic, a fatal runtime error) are written to the same file, so a crash leaves its reason next to the configuration instead of only a "terminated unexpectedly" event. **A Windows service has no console**, so the MSI's default configuration sets `C:\ProgramData\logrok-universal-agent\agent.log`; on Linux/macOS leave it empty and read the journal / launchd logs. An unusable path logs a warning and falls back to stdout — the agent never goes silent |
| `fips_mode` | string | `auto` | ✅ | **A check, not a switch.** `auto` (default) has no opinion · `required` refuses to start unless the binary is running in FIPS 140-3 mode · `disabled` refuses to start if it is. FIPS mode is selected when the binary is **built** and locked at process start, so no configuration can turn it on or off — what this buys is catching the **wrong artifact**. The FIPS and standard builds are identical in every observable way except the crypto module, so without this a regulated deployment can only discover it shipped the wrong one during an audit. `disabled` is not symmetry for its own sake: a FIPS build refuses non-approved cipher suites, so an estate that must reach a legacy receiver needs to fail at startup rather than at the first delivery. See [FIPS 140-3 build](USER-GUIDE.md#fips-140-3-build) |
| `fips` | string | *(unset — run as built)* | ✅ | **Sets** the cryptographic posture (`fips_mode` only *checks* it). `on` or `off`; unset runs the binary as built, and the shipped build is FIPS. An explicit `on` on a non-FIPS build is a startup error, not a silent no-op. Applied at startup — **the process ID does not change**, so service supervision is unaffected. Startup only: changing it needs a restart, not a reload. Turn it **off** only for a receiver that offers neither TLS 1.3 nor Extended Master Secret, which FIPS 140-3 requires for TLS 1.2 — Splunk HEC is the one we know of. **Windows:** the setting cannot be applied this way; set `GODEBUG=fips140=off` in the service environment instead; a disagreeing config value is reported, not ignored. See [Turning FIPS off](USER-GUIDE.md#turning-fips-off-for-a-receiver-that-cannot-do-it-servicefips) |

**Metrics exposed:** `logrok_agent_events_in_total`, `events_out_total`, `events_dropped_total`,
`backpressure_waits_total`, `spool_meta_write_errors_total`, `spool_write_errors_total`,
`udp_truncated_total`, `spool_tamper_detected_total`,
`pipeline_stage_busy_seconds_total{stage}`, `pipeline_stage_wait_seconds_total{stage}` (counters per part of the pipeline; `stage` is
`intake`, `parallel`, `serial` or `ship`, counted only with `service.workers` above 1),
`pipeline_hop_events_total{hop}`, `pipeline_hop_busy_seconds_total{hop}`,
`pipeline_hop_wait_in_seconds_total{hop}`, `pipeline_hop_wait_out_seconds_total{hop}` (counters per hop of the
path: the four stages above, `dest:<name>` for a destination's delivery, `spool:<name>` for writes to its
spool and `drain:<name>` for delivering its spool backlog; busy is time working, wait-in time idle waiting
for work, wait-out time held back by the next hop; events ÷ busy seconds is the rate the hop could reach if
it never waited, and the hop with the lowest such rate limits throughput),
`pipeline_workers`, `pipeline_parallel_prefix`, `pipeline_inflight_batches`, `pipeline_reorder_depth`,
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

**Delivery timing and reduction funnel (since 1.4.0):** `output_event_delay_seconds` — largest gap, in the
most recently delivered batch, between an event's own timestamp and the moment a destination accepted it;
how stale the newest data reaching your SIEM is (includes upstream lag and spool wait; a skewed source clock
shows as noise). `pipeline_latency_seconds` — largest time, in that batch, from an input handing an event to
this agent until a destination accepted it; the agent's own queueing and spool wait, independent of source
clocks (events spooled by a pre-1.4 binary are not counted). Both have per-destination variants
(`output_<name>_event_delay_seconds`, `output_<name>_pipeline_latency_seconds`; on `/metrics` the labelled
families `logrok_agent_output_delivery_*`) and read 0 until the first delivery. With several destinations the
unlabelled keys report the most recent delivery by **any** destination — not a sum and not a maximum — so use
the per-destination variants when you fan out. `processor_<name>_dropped_total`
— events removed by each named processor (with `events_in_total` and `events_out_total`: the per-stage
funnel). `processor_<name>_overflow_total` — events a capped processor (`aggregate` at `max_groups`) passed
through un-reduced; nothing is lost, but the reduction did not apply to them. `processor_reduction_ratio` — processor drops ÷ events in, 0–1, since start, including events that matched
no destination's `when:` condition, which appear as `processor_routing_when_dropped_total`; spool `when_full` drops
are excluded (loss, not reduction — they stay in `events_dropped_total`). `fips140_enabled` /
`fips140_ems_relaxed` — 0/1 posture gauges, also in every heartbeat. `snmptrap_legacy_crypto_users` — SNMPv3 users on md5 auth / des privacy (present while a v3 `snmptrap_in` listener runs). The `/metrics` HELP text for each key
carries the full description these paragraphs summarise.

**Why events were lost, and how old the backlog is (since 2.0.0):** every event counted in
`events_dropped_total` is also counted under exactly one reason, so the reasons always add up to the total —
on `/metrics` as `logrok_agent_events_dropped_by_reason_total{reason="…"}`, in the heartbeat as
`events_dropped_by_reason_<reason>_total`. The reasons: `output_rejected` (the destination refused the batch
permanently and it was dropped or dead-lettered — fix the payload or the receiver), `spool_full` (the spool
reached `max_bytes` under a drop policy — raise it, or use `when_full: block`), `disk_low` (the spool's disk
was below `buffer.min_free_bytes` under a drop policy), `no_spool` (the destination was down and no spool is
configured), `spool_withdrawn` (the destination was down while the spool only drains — see "If
store-and-forward is withdrawn" below), `shutdown` (the agent stopped before it could deliver or durably spool
the events — waiting for spool space under `when_full: block`, or with a last batch a destination could not take
and no disk spool to keep it; file, journal and event-log inputs read these events again after the restart), `input_overflow` (an input's own queue overflowed before the event entered the pipeline —
inputs whose source cannot be slowed down: `snmptrap_in`, `modbus_in`, `etw`; raise that input's
`queue_size`/`buffer_events` or reduce the source rate). Inputs with their own counters also report them as
`input_<name>_<counter>` (on `/metrics`: `logrok_agent_input_<counter>{input="<name>"}`), where `<name>` is the
input's `name` or else its type: `queue_dropped_total` (`snmptrap_in`, `modbus_in`), `rejected_version_total`
and `rejected_community_total` (`snmptrap_in`), `polls_total`, `poll_errors_total` and `events_total`
(`modbus_in`). `oldest_unsent_age_seconds` is the age of the oldest event still waiting in a spool (the
largest across destinations; per destination `output_<name>_oldest_unsent_age_seconds`), 0 when every spool
is empty — unlike `output_event_delay_seconds` it keeps growing while a destination is down. Under
`when_full: block`, `output_<name>_backpressure_waits_total` names the destination that is slowing your
sources; `output_<name>_held_commit_overflow_total` counts events whose input position moved on early because
that destination's in-memory spool already had 65,536 events waiting to be confirmed (see "Use a disk spool
with routing"); `spool_disk_low_total` counts spool writes refused for `min_free_bytes`. The resource caps report
`gomaxprocs` always, and `memory_limit_bytes`, `memory_limited` (1 while inputs are paused) and
`memory_limited_total` when `limits.memory_limit_mib` is set — see [`limits`](#limits).

<a id="enroll"></a>
## `management`

Central management (a non-empty `endpoint`) and enrolled mTLS (`tls.mode: enrolled`) are **Apex** capabilities. On an
unlicensed agent they keep running, but the agent reports posture `degraded`.

Central control plane. **Empty `endpoint` = unmanaged** (runs purely from the local file — valid for air-gapped
installs). Auth uses the logrok control-plane headers: `X-Api-Key` + `X-Tenant-Slug`. Per-agent **mTLS** hardens the channel on top of the api-key: set `tls.cert_file`/`tls.key_file` to present an operator-provisioned client certificate (`tls.mode: static`), or use `tls.mode: enrolled` to have the control plane issue and auto-renew a per-agent client certificate.

| Key | Type | Default | Status | Notes |
|---|---|---|---|---|
| `endpoint` | string | `""` | ✅ | `""` = unmanaged. Use HTTPS in production — the api-key is a bearer credential |
| `enrollment_token` | string | `""` | ✅ | **one-time** secret; exchanged at first start for `agent_id` + `api_key` (retried with backoff). A first start waits up to 30 s for it before reading, so the first events carry `agent_id`; past that, collection runs regardless. Credentials persist in `state_path`, so the token can be removed after enrollment |
| `endpoint` value `auto` | string | — | ✅ | **discovery**: browse the local segment (mDNS/DNS-SD `_logrok-cp._tcp.local`) for a control plane instead of hard-coding a URL. Opt-in only — an agent with an endpoint written down is never re-pointed by the network. Discovery **locates, it never authenticates**: the enrollment token and TLS still prove the control plane, so a rogue advertiser only gets ignored. Finding nothing is not an error: the agent runs unmanaged for that run and says so. |
| `enrollment_bundle` | string | `""` | ✅ | path to a signed **air-gap enrollment bundle** (identity + entitlement + optional starting config), verified offline against the embedded issuer key. Adopted once (anchored by the bundle's own id) and then inert — safe to leave the media mounted. See [Enrolling an agent with no path to the control plane](USER-GUIDE.md#enrolling-an-agent-with-no-path-to-the-control-plane-bundles). |
| `state_path` | string | `$STATE_DIRECTORY/agent-state.json` when the service manager names a state directory (systemd `StateDirectory=`, the FreeBSD rc script, the container image), else `agent-state.json` next to the config | ✅ | where enrollment credentials + applied config version persist (atomic, `0600`). Secrets live here, never in the config file. On Windows, give it (and every other agent-level path) a drive letter: a configuration pushed from the control plane with a drive-less path such as `/usr/local/…` is refused |
| `report_config` | bool | `true` | ✅ | report the running configuration to the control plane on every heartbeat (its hash always; the text when the control plane holds a different one) so the fleet view can show it and adopt it as the managed configuration. **Secrets never leave the host**: every secret value (any key whose name contains `token`, `password`, `passwd`, `secret`, `api_key`, `apikey`, `passphrase`, `private_key` or `credential`, plus `Authorization`-style keys under `headers`) is replaced by `${logrok:keep}` before the text is hashed or sent; file paths and identifiers stay readable; comments and key order survive, formatting is normalized. A configuration pulled from the control plane may carry that same marker under a secret key — the agent then keeps its current local value, and refuses the whole configuration if it has none or if the marker sits under a non-secret key (the refusal is reported on the next heartbeats as a failed apply). Multi-document YAML files are neither reported nor adoptable. `false` sends nothing. See [What the control plane can see of your configuration](USER-GUIDE.md#what-the-control-plane-can-see-of-your-configuration). |
| `api_key` | string | `""` | ✅ | manual provisioning alternative to enrollment; sent as `X-Api-Key` (enrolled state takes precedence) |
| `remote_upgrade` | bool | `true` | ✅ | let the control plane upgrade this agent from its Agents page (see [Upgrading agents from the control plane](USER-GUIDE.md#upgrading-agents-from-the-control-plane)). The agent installs a binary only when (1) the offer comes from the enrolled control plane — signed by it, or unsigned over an https connection to it (plain http: signed only) — names this agent, its platform and CPU architecture, and has not expired; (2) the downloaded file's checksum matches the offer **and** appears in a `SHA256SUMS` signed with the logrok **release** key, verified on the host against the key built into the agent — a control plane cannot sign binaries; and (3) the new binary, run once out of process against this configuration, reports the offered version. Nothing is touched before those pass; every failure is reported on the next heartbeat with its reason and the running binary is untouched. `false` refuses every offer (reported as refused) — for hosts upgraded by hand or by package management only |
| `upgrade_dir` | string | `$STATE_DIRECTORY/upgrade`, else `upgrade/` beside `state_path` | ✅ | **Linux/macOS/BSD/Solaris:** where a remotely delivered binary is staged. The agent never replaces the file your package manager installed — it runs the staged binary from here (re-verified against the release key at every start) and falls back to the packaged one when that is newer or the staged one fails verification. Must be writable by the agent's user; the systemd unit's `StateDirectory` is. Delete this directory to roll back to the packaged binary. Unused on Windows, where the service replaces its own executable (the previous one is kept as `logrok-universal-agent.prev.exe`) |
| `tenant_slug` | string | `""` | ✅ | sent as `X-Tenant-Slug` |
| `heartbeat_every` | duration | `30s` | ✅ | heartbeat: agent_id, hostname, platform, version, config version, events/min, buffer depth |
| `config_poll_every` | duration | `60s` | ✅ | config-pull cadence (ETag/If-None-Match — unchanged config costs a 304) |
| `tls.mode` | string | `static` | ✅ | `static` (operator-provided cert files) or `enrolled` (the control plane issues and the agent **auto-renews** a per-agent client certificate — requires `endpoint` + enrollment, and a control plane with the agent-cert CSR endpoint). The key/cert/chain persist next to `state_path` as `agent-cert.pem`/`agent-key.pem`/`agent-ca.pem` (`0600`); the private key never leaves the host. Applies to both the **control-plane** channel and **data-plane** outputs that opt in with `cert_source: enrolled` |
| `tls.ca_file` | string | `""` | ✅ | PEM CA bundle to verify the control-plane server. **Static mode:** set = only this CA is trusted (pinned); empty = system roots. **Enrolled mode:** added to the trust pool together with the CA chain the control plane delivers and, unless `tls.system_roots: false`, the system roots |
| `tls.system_roots` | bool | `true` | ✅ | **Enrolled mode only.** Also trust the host's system certificate store for the control-plane server, so a control plane with a certificate from a public or corporate CA keeps verifying after enrollment without `ca_file`. `false` pins trust to `ca_file` plus the CA chain the control plane delivered. Upgrade offers without a signature are accepted only over a verified https connection, so this matters for remote upgrade too |
| `tls.cert_file` | string | `""` | ✅ | client certificate (PEM) presented to the control plane for **mTLS** (set with `key_file`) |
| `tls.key_file` | string | `""` | ✅ | client private key (PEM) for mTLS |
| `tls.server_name` | string | `""` | ✅ | SNI / verification name override |
| `tls.insecure_skip_verify` | bool | `false` | ✅ | **dev only** — disables server verification. While it is `true` the agent **refuses upgrade offers without a signature** (reported as `refused`, naming this setting): with verification off, the connection no longer proves an offer comes from your control plane. Signed offers are still accepted |
| `tls.max_cert_age` | duration | `0` (off) | ✅ | **Enrolled mode only.** Renew the certificate once it reaches this age, even though it is far from expiry. Set this if your control plane rotates its agent CA **and issues long-lived certificates**. A rotation runs with an overlap period during which both the old and new roots are trusted, and an agent moves onto the new root **by renewing**. An agent whose certificate does not reach its normal renewal window during the overlap never moves; when the operator completes the rotation, the certificate it presents still chains to the retired root and the receiver rejects it. Nothing looks wrong until that moment. **Size it below two-thirds of your issued certificate lifetime** — that is where ordinary renewal already fires, so anything at or above it never takes effect (a value equal to the lifetime is a guaranteed no-op: the certificate expires before it can get that old). Costs one extra issuance per interval per agent. Left off by default because against a control plane issuing short-lived certificates the ordinary window already moves every running agent well inside a normal overlap. |
| `config_signing_keys` | []string | `[]` | ✅ | pin one or more Ed25519 public keys (`ed25519:<base64>`, as printed by `-gen-config-signing-key`). When set, a configuration pulled from the control plane is applied **only if** it carries a valid signature, by one of these keys, over its exact text (`X-Config-Signature` on the pull response); an unsigned or wrongly signed configuration is refused, the running one stays, and the refusal is reported on the next heartbeats. The control plane never sees the private key, so even a compromised control plane cannot push a configuration you did not sign. Set it in the local configuration. A pulled configuration may **rotate** the keys (it must itself be signed by a current one) but may never **remove** them — a signed configuration without `config_signing_keys` is refused, so turning enforcement off is always a local change on the host. Not covered: re-sending an older configuration you signed earlier, or one you signed for a different group of agents — a signature covers the text, not where it is sent; rotate the key when an old signed configuration must never apply again. Requires a control plane that stores and serves the signature. Since 2.0.0 |

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
| `enabled` | bool | `false` | ✅ | turn the spool on — **required**: setting `dir` (or any other buffer option) alone does not enable it, and the agent warns at startup when buffer options are set but `enabled` is not |
| `dir` | string | `""` | ✅ | spool directory. **Set this** to survive restarts; empty → in-memory only. An in-memory backlog is not kept across a restart, but its events are acknowledged to their inputs only once delivered (or dropped by policy), so a file, journal or acknowledging network input sends them again after the restart (possible duplicates, no loss). While the output is down, those inputs' positions and acknowledgements therefore wait at the oldest undelivered event. |
| `max_bytes` | int | `0` | ✅ | on-disk cap; `0` = unbounded. Behaviour at the cap is set by `when_full`. Honored for any positive value; a very small cap (below ~64 KiB) may transiently hold up to one ~64 KiB segment. |
| `when_full` | string | `drop_oldest` | ✅ | `drop_oldest` (discard oldest spooled, keep freshest) \| `drop_newest` (refuse new, keep oldest) \| `block` (back-pressure the source until space frees — no loss). |
| `flush_every` | duration | `2s` | ✅ | how often each destination's spool drain loop wakes to move its backlog once the output is reachable again. The live path ships immediately — this paces **recovery**, not delivery. While writes keep failing, the retries back off: each consecutive failure doubles the wait, up to 30 s (never less than `flush_every` itself), with ±20% jitter so many agents do not retry in step; the first success resets it. Until 1.3.1 the key was accepted and validated but not read (the loop ran at a fixed 2 s); a non-default value now takes effect |
| `chain_key_file` | string | `""` (auto-resolved) | ✅ | overrides where the tamper-evidence chain key lives. Empty = resolved automatically (see below). |
| `min_free_bytes` | int | `0` | ✅ | keep at least this many bytes free on the disk holding the spool; `0` = no floor. The spool never writes past it, whatever `max_bytes` allows, and other programs' use of the same disk counts. A batch that would cross it follows `when_full`: `block` waits (back-pressure, no loss), `drop_newest` drops the batch, `drop_oldest` first discards the oldest spooled events to make room and drops the new batch only when nothing is left to discard. Counted in `spool_disk_low_total` and, when events are lost, under reason `disk_low`. Disk spool only (`dir` set); on a platform where free space cannot be read the agent logs one warning and the floor is not applied. Since 2.0.0 |
| `encrypt` | bool | `false` | ✅ | encrypt every new spool record at rest with AES-256-GCM (FIPS 140-3 approved), so a copy of the spool directory reveals no event content. The key lives outside the spool (`encryption_key_file`). Turning it off later writes new records in clear and still reads the encrypted ones while the key file exists. If the key is missing or replaced, encrypted records are **kept and delivery pauses** — an error in the log and `spool_decrypt_stalled` above 0 — until the key is restored; they are never discarded. A record that was **modified** after it was written — the tamper chain shows it, under a chain key that verifies the segment — is delivered as a placeholder event flagged `spool_tamper` and `spool_decrypt_failed`, carrying the stored ciphertext in `spool_sealed_record` for later examination, so a forged record can neither be read nor block the spool. When the keys themselves are the problem (for example the key directory was lost and new keys were created), every record stays in the spool and delivery pauses instead. Disk spool only (`dir` set). Since 2.0.0 |
| `encryption_key_file` | string | `""` (auto) | ✅ | where the encryption key lives. Empty = `spool-encryption.key` beside the chain key (never inside the spool directory), created on first use with owner-only permissions. **Back it up**: without it, encrypted records cannot be delivered. With `encrypt: true`, a key that cannot be read or created stops the agent from starting rather than writing in clear |

On disk: every spooled record is checksummed and written durably (fsync) before it counts as spooled. A torn
or bit-rotted record fails its checksum and is skipped on replay (the drain doesn't stall); a torn tail from a crash mid-append is truncated on
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
chain** that links each record, and each spool file, to the one before it. This is **on automatically** — there is no
config toggle to turn it off (short of the key becoming unavailable, below) — so the claim "every deployed
agent's spool is tamper-evident" holds fleet-wide without per-agent setup.

- **Key.** 32 random bytes, hex-encoded, file mode `0600`, auto-generated on first use if absent. Resolution
  order (shared byte-for-byte between the running agent and the `-verify-spool` CLI so the two can never disagree about where the key is):
  1. `buffer.chain_key_file`, if set;
  2. next to `management.state_path` as `spool-chain.key`. When the agent loads a configuration file,
     `state_path` is always set (it defaults to `agent-state.json` in the service state directory, else
     beside the configuration file), so this is the normal location for managed and standalone agents alike;
  3. only when no `state_path` is set at all, a sibling of the spool `dir`, named `<dir>.chain.key`
     (e.g. `dir: /var/lib/agent/spool` → `/var/lib/agent/spool.chain.key`).

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
  error-level log line, the `spool_tamper_detected_total` metric (above), and a **sticky** heartbeat
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

## `limits`

Caps on the agent's own use of the host — so a burst or a stuck destination can't take memory or CPU away
from what the host is for. Every key is optional; unset means no cap. Not a licensed capability. Since 2.0.0.

| Key | Type | Default | Status | Notes |
|---|---|---|---|---|
| `max_procs` | int | `0` | ✅ | the most CPU cores the agent runs on at once. `0` = the Go runtime default: every core, or the container's CPU quota when one is set. This caps parallelism, not a percentage — `1` on an 8-core host bounds the agent to one core's worth of CPU. Reported as `gomaxprocs` |
| `memory_limit_mib` | int | `0` | ✅ | the agent's memory budget, in MiB (`0` = none; otherwise at least `32`). The agent's garbage collector works to stay under 80% of it. At **90%** the agent stops reading from its inputs and resumes once memory is back **under 80%** — events it has already accepted are kept, and spools keep draining meanwhile. While paused, sources are held back the same way `when_full: block` holds them: file and journal inputs simply read later; TCP senders are slowed by flow control; UDP datagrams that arrive while paused can be lost by the operating system, outside the agent's counters. Each pause is logged (`memory limit reached — inputs paused`) and reported as `memory_limited` / `memory_limited_total`. The limit applies to the memory the agent's runtime holds in use; the process's resident size (`rss_bytes`) is larger by the program's own code and by memory not yet handed back to the operating system — measured with a 64 MiB limit: resident size peaked at about 96 MB during a pause and fell to 26 MB once the backlog drained. Set it well above the agent's normal footprint — it is a safety net for bursts and long outages with an in-memory spool, not a target |

Both caps take effect on reload; removing one restores the runtime default. A `GOMAXPROCS` or `GOMEMLIMIT`
set in the service environment is left alone unless the matching key is set here.

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

### Modules marked Apex in 1.2.0 — enforced since 2.0.0

The modules below are Apex, and are marked **Apex** in their reference sections above. Releases before
1.2.0 did not mark or check them, and 1.2.0 through 1.x only warned about them, so a configuration built
on 1.x may be using them without an entitlement:

| | Modules |
|---|---|
| Inputs | `etw` · `wmi` · `linux_audit` · `mqtt_in` |
| Processors | `redact` · `lookup` · `adaptive_sample` |
| Outputs | `snare` · `hec` · `sentinel` · `xsiam` · `kafka` · `s3` · `loki` · `otlp` · `elasticsearch` · `beats` · `fluent` (`elasticsearch`, `beats` and `fluent` added in 2.0.0, same terms) |

**Since 2.0.0 they are enforced.** If your entitlement does not cover them, the agent removes them at
start (logged as `licensing: degraded to Core — unlicensed Apex features disabled`, naming each module),
reports posture `degraded`, and keeps running everything else. Before upgrading from 1.x, add a license
(`license_file`), enroll with a logrok control plane, or change the configuration.

The free **Core** tier is unchanged and is exactly what it has always been: file tail, syslog-in,
journald, HTTP-in, the parsers (`parse_json`/`parse_csv`/`parse_kv`/`parse_xml`/`expr`/`add_fields`/
`filter`), RFC 5424 syslog output over TCP/TLS, and `/metrics`. Nothing was taken out of Core. Core
forwarding is never blocked in any posture, and TLS is never gated.

To check a configuration before upgrading, start the agent and read the startup log: a `licensing:`
warning naming modules is the notice; no such line means the configuration is unaffected.

---

## `inputs`

**Options every input takes** (beside its own):

| Option | Type | Default | Notes |
|---|---|---|---|
| `add_fields` | map | `{}` | fields stamped on every event of **this input** (before the processors; values are strings, numbers or booleans — a nested map or list is refused). The per-input form of the [`add_fields`](#add_fields-) processor, which stamps every input's events. Overwrites a field of the same name the input set |
| `host_override` | string | `""` | replaces the host of every event of this input (the syslog HOSTNAME, the HEC `host`) |

Both are refused on processors and outputs. A converted Splunk forwarder uses them to carry each stanza's
metadata: see [Splunk metadata fields](#splunk-metadata-fields).

```yaml
- type: filetail
  paths: ['/var/log/nginx/**']
  host_override: web-01
  add_fields:
    splunk.index: web
    splunk.sourcetype: "nginx:access"
```

<a id="saved-read-positions"></a>
#### Saved read positions

Inputs that can resume after a restart **save their read position by default**: `filetail` (`state_path`),
`journald` (`cursor_file`), `oslog` (`checkpoint_file`) and `linux_audit` in file mode (`state_path`). When
the option is left out, the agent keeps the position in a file of its own under the `positions/` directory
beside [`management.state_path`](#management) — so under the service's state directory
(`$STATE_DIRECTORY/positions/` with the packaged systemd unit, the FreeBSD rc script and the container
image), else `positions/` next to the configuration file. The directory is created owner-only (`0700`),
each file `0600`, written atomically.

- **Option left out** → `positions/<type>-name-<name>.<ext>` when the input has a `name`, else
  `positions/<type>-<hash>.<ext>`, where the hash is taken from what the input reads (`filetail`'s
  paths, `journald`'s units/identifiers/priority, `oslog`'s predicate/level, `linux_audit`'s path). Never
  the input's place in the list, so reordering inputs keeps every position. Changing what an unnamed
  input reads gives it a new file (it starts again per its `start_at` / `read_from`); give the input a
  `name` to keep its position across such edits.
- **Option set to `""`** (or left empty) → no saved position: every start applies `start_at` / `read_from`,
  as before.
- **Option set to a path** → that file, unchanged.

Two inputs never share a default file: when two inputs would get the same one (same type, no name, the
same paths), the later one gets a numbered suffix and the agent logs a warning asking for distinct names.
Two inputs whose **explicit** paths name the same file load with a warning too: each rewrites the whole
file, so give each its own path.

**Upgrading.** An installation whose configuration leaves the option out has no saved position on its
first start after the upgrade, so `start_at` / `read_from` applies once, as before; from then on the
position is saved and every restart resumes from it — with `start_at: end`, records written while the
agent was stopped are now delivered instead of skipped. `windows_eventlog` is unchanged: its
`bookmark_path` has always defaulted to `C:\ProgramData\logrok-universal-agent\bookmarks`.

If the directory is not writable (for example a read-only configuration directory and no state
directory), collection continues and the input logs one warning per failing streak; set
`management.state_path` or the input's option to a writable path. `-check-config` and the agent's
startup report this up front (read positions will not be saved: the directory is not writable by this
process) when an input uses a default position file; the check never creates anything, and on Windows
it cannot tell and stays silent.

#### Splunk metadata fields

Four event fields carry Splunk metadata per event: `splunk.index`, `splunk.sourcetype`, `splunk.source`,
`splunk.host` (plus `splunk.route`, a converted forwarder's per-input routing, as `,group1,group2,`).

- The `hec` output (see [`outputs`](#outputs)) uses them for that event's `index` / `sourcetype` / `source` / `host`, ahead of its own static
  `index` / `sourcetype` / `source` and the event's host. **No `splunk.*` field is ever sent as an indexed
  field.**
- Every other output carries them like any field — the `syslog` output as RFC 5424 structured-data
  parameters (a logrok receiver stores them nested: `extra.splunk.sourcetype`).

<a id="eventlog"></a>
### `windows_eventlog` ✅ *(verified end-to-end on real Windows hardware)* — Windows only — **Apex**

**Apex:** an unlicensed configuration has this input removed at load.

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
| `resolve_sids` | bool | `true` | resolve the event's user SID (`<Security UserID>`) to `account_name` / `account_domain` / `account_type` on the collecting host. Each SID is looked up once and cached; turn it off where a lookup would wait on an unreachable domain controller |
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

The rest of `<System>` is emitted too (NXLog `im_msvistalog` field parity), each only when the event carries it:

| Field | From | NXLog name |
|---|---|---|
| `event_version` | `<Version>` | `Version` |
| `task_id` / `task_name` | `<Task>` and its name from the provider's manifest | `Task` / `Category` |
| `opcode_id` / `opcode_name` | `<Opcode>` and its name | `OpcodeValue` / `Opcode` |
| `keywords` / `keyword_names` | `<Keywords>` (hex, as in the event XML) and the keyword names, comma-separated (`Audit Success`, `Classic`) | `Keywords` (decimal) |
| `provider_guid` | `<Provider Guid>` | `ProviderGuid` |
| `user_sid` | `<Security UserID>` | `UserID` |
| `activity_id` / `related_activity_id` | `<Correlation ActivityID / RelatedActivityID>` — groups the events of one activity | `ActivityID` / `RelatedActivityID` |
| `account_name` / `account_domain` / `account_type` | the SID resolved on the collecting host (`resolve_sids`); type is `User`, `Group`, `WellKnownGroup`, `Alias`, `Computer`, … | `AccountName` / `Domain` / `AccountType` |

These names are deliberately not the bare XML element names: `Version`, `Task` and `Opcode` are common
`EventData` names (Sysmon event 4 carries `Version`), and those keep their own names unchanged.

**Reserved-field protection.** All the `<System>`-derived names above are reserved. If an `EventData`/`UserData`
entry uses one of them as its data name (case-insensitively — e.g. `<Data Name="event_id">` or
`Name="Channel"`), it does **not** overwrite the trusted value; instead it is preserved under an
`eventdata_`-prefixed key (`eventdata_event_id`, `eventdata_Channel`, …). This keeps a record from forging the
exact fields detections key on (spoofing `event_id=4624` while the real EventID differed, or faking
Channel/Provider/the account). The original casing of the data name is preserved after the prefix.

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

<a id="eventlog-wec"></a>
**Windows Event Forwarding (WEF):** if you already run a WEF collector (WEC), its `ForwardedEvents`
channel is collectible the same way — `channels: [ForwardedEvents]` on the WEC host drains the whole
WEF estate through one agent. (Acting as a WEC *server* is out of scope; per-endpoint agents replace WEF.)

### `etw` ✅ *(runtime-verified on real Windows hardware)* — Windows only — **Apex**

Consumes **ETW (Event Tracing for Windows)** providers through an agent-owned **real-time trace session** —
the high-volume kernel/analytic telemetry that never reaches the Event Log (kernel process/network activity,
DNS Client analytic tracing, Time/Update service traces; the NXLog `im_etw` engine gap). Uses the native
Windows trace API — still one static binary, nothing extra to install. On non-Windows builds this is a stub that validates config but
errors if started.

| Option | Type | Default | Notes |
|---|---|---|---|
| `providers` | []string | *(required)* | providers to enable — each a **GUID** (`{xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}`, braces optional) or a registered **manifest provider name** (e.g. `Microsoft-Windows-DNS-Client`), resolved at start via `TdhEnumerateProviders`. A name that doesn't resolve is a start error telling you to use the GUID (`logman query providers` lists both) |
| `session_name` | string | `logrok-agent-etw` | the trace-session name the agent owns. A stale same-named session left by a previous crash is stopped and reclaimed automatically at start; the session is always stopped on shutdown |
| `level` | int \| string | `4` | max event verbosity: `1`–`5` or `critical`/`error`/`warning`/`info`/`verbose` |
| `match_any_keyword` | int \| string | `0` (all) | provider keyword mask (`EnableTraceEx2` MatchAnyKeyword); accepts an integer or a hex string like `"0x10"` — use the hex-string form for high-bit masks that overflow YAML integers |
| `buffer_events` | int | `4096` | events buffered while the pipeline catches up (see the drop note below) |

**Privileges (required).** Real-time ETW sessions need **Administrator** or membership in the
**Performance Log Users** group. Without either, the input fails at start with an explicit
"requires Administrator or Performance Log Users membership" error — there is no degraded mode.

**Backpressure = count-and-drop (documented behavior).** ETW cannot be slowed down: holding events back
would make **ETW itself** drop them silently and without any count. So when the pipeline can't keep up and the internal `buffer_events` buffer is full,
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
a real-time trace (`etw`). It requires **PowerShell**, which is present on every supported Windows, and runs
it once per poll. On non-Windows
builds this is a stub that validates config but errors if started.

| Option | Type | Default | Notes |
|---|---|---|---|
| `query` | string | *(required)* | the **WQL** query, e.g. `SELECT Caption,Version FROM Win32_OperatingSystem`. Single quotes inside it (e.g. `WHERE State='Running'`) are kept as written |
| `namespace` | string | `root/cimv2` | the WMI namespace to query (most classes live in `root/cimv2`; some newer ones in `root/StandardCimv2`, etc.) |
| `interval` | duration | `60s` | how often to run the query. WMI is a *poll*, not a stream — each interval re-runs the query and emits the current result set |
| `source` | string | `wmi` | the `Event.Source` label for emitted events |
| `facility` | int | `16` | syslog facility (a WMI instance has none); `16`=local0, matching eventlog/etw/journald |
| `severity` | int | `6` | syslog severity (`6`=info) |

Each returned CIM instance becomes one event; each of its properties becomes a field (flattened one level —
a scalar becomes its string form; a nested object or array is sent as compact JSON, nested at most 3 levels
deep). Bookkeeping properties that carry no telemetry (`PSComputerName`, `PSShowComputerName`, `CimClass`,
`CimInstanceProperties`, `CimSystemProperties`) are dropped. Field keys are kept verbatim; a would-be collision is
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

<a id="filetail"></a>
### `filetail` ✅ — cross-platform

Tails plain-text log files, emitting one event per line — the universal input for anything that writes a
log file: application logs, IIS/web-server logs, appliance exports, Kubernetes container logs. It follows
globs, survives rotation (rename or in-place truncation) and agent restarts (offset checkpoints via
`state_path`, saved by default), assembles multiline records such as stack traces, and decodes UTF-16 (the Windows norm for
PowerShell redirects and `wevtutil` exports). If a program logs to a file, this is the input to reach for.

| Option | Type | Default | Notes |
|---|---|---|---|
| `path` | string | `""` | a single file or glob to tail (`path` or `paths` required) |
| `paths` | []string | `[]` | one or more globs. Supports `*` `?` `[…]` (within a path segment) and recursive `**` (whole segment, e.g. `/var/log/**/*.log`) |
| `exclude` | []string | `[]` | globs to skip (same syntax) |
| `start_at` | string | `end` | `end` (new lines only) or `beginning` (whole file) — applies only when there is no saved checkpoint for the file |
| `state_path` | string | `positions/filetail-<key>.json` beside `management.state_path` | offset-checkpoint file for resume-after-restart (see [Saved read positions](#saved-read-positions)). `""` = no persistence (every restart re-applies `start_at`) |
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

`format: text | cri | docker | auto` (default `text`). Container-log mode — any value other than `text` — is
**Apex** (an unlicensed agent removes the whole input); plain text tailing is Core. When set to `cri`/`docker`/`auto`,
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
RFC 5424 and RFC 3164 over TCP and/or UDP, over TLS (RFC 5425, with optional client-certificate verification), and
(on unix platforms) a local unix-domain socket as a `/dev/log` replacement.

| Option | Type | Default | Notes |
|---|---|---|---|
| `listen` | string | `""` | `host:port`, e.g. `0.0.0.0:5514` (at least one of `listen`, `tls_listen` or `socket` required) |
| `protocols` | []string | `[tcp, udp]` | any of `tcp`, `udp` (applies to `listen`) |
| `socket` | string | `""` | **unix platforms**: receive local syslog on a unix-domain socket (NXLog `im_uds` parity) — a `/dev/log` replacement on hosts with no syslog daemon. A stale socket file from a crash is replaced; a non-socket file at the path is **refused**, never deleted |
| `socket_protocol` | string | `unixgram` | `unixgram` (datagram — what `syslog(3)`/`logger` speak) or `unix` (stream) |
| `socket_mode` | string | `0666` | octal permission bits on the socket file (default lets unprivileged local writers log) |
| `tls_listen` | string | `""` | `host:port` for **syslog over TLS** (RFC 5425), conventionally `0.0.0.0:6514` — served beside (or instead of) `listen`, with the same parsing and `framing` |
| `tls_cert_file` / `tls_key_file` | string | `""` | the server certificate and key for `tls_listen` (both required with it) |
| `tls_client_ca_file` | string | `""` | **mTLS**: require and verify client certificates on `tls_listen` against this CA |
| `framing` | string | `auto` | how TCP and unix-stream connections are split into messages: `auto` (RFC 6587 octet-counting when a frame is `LEN SP <PRI>…`, otherwise one message per line), `newline` (always one message per line — for raw text streams from applications or forwarders that send plain lines) or `octet-counting` (RFC 6587 only; anything else closes the connection). UDP and unix datagrams are one message per datagram regardless |
| `parse_tag` | bool | `false` | **RFC 3164 only**: split the leading TAG (`myapp[88]: text`) off the message — the program name becomes the `app` field, a bracketed process id the `procid` field, and the message keeps only the text after `: `. The syslog output then sends them as APP-NAME and PROCID. A message that only looks like a tag is left unchanged: the tag must be 1–32 printable characters without `[`, `]`, `:` or a space, an optional non-empty `[pid]`, then `:` followed by a space or the end of the message (after `[pid]` the space may be missing) — so `http://x`, `disk full: /var` and `key:value` stay as they are. Also recognises a local sender's message that carries no hostname (`syslog(3)` on the socket). Off by default, so existing message content does not change |
| `max_connections` | int | `1024` | concurrent stream connections, counted across `listen` (TCP), `tls_listen` and a `unix` stream socket together; a connection beyond it is closed at once and logged as a warning. UDP and unix datagrams are not connections and are not limited |
| `udp_listeners` | int or `auto` | `1` | UDP sockets listening on `listen`, each read on its own core (1-64; `auto` = one per available core). Above 1, datagrams on the port are spread over the listeners, so even a single busy sender (a relay, a firewall) uses several cores. **Message order across listeners is not kept**: with 1 (the default) order is exactly as received. Only Linux and FreeBSD support a value above 1: elsewhere it is refused and `auto` means 1. Needs no extra privileges |
| `udp_receive_buffer_bytes` | int | `8388608` (8 MiB) | receive buffer each UDP socket asks the kernel for; `0` keeps the OS default (~208 KB on Linux), which a burst overflows and the kernel then drops datagrams. The kernel caps the request at `net.core.rmem_max`; when it grants less, the agent logs a warning with both values |

Parses RFC 5424 (PRI→severity/facility, timestamp, host, `app`/`procid`/`msgid`; **structured-data exploded into
fields** — each `PARAM` becomes a field plus a joined `sd_id`) and RFC 3164 (PRI + **`Mmm dd HH:MM:SS` timestamp
and hostname extracted**, tag+body kept as the message unless `parse_tag: true`; assumes the current year). TCP auto-detects RFC 6587
octet-counting vs newline framing (`framing: auto`); a line that merely starts with digits — a timestamp such as
`2026-10-16 12:00:01 …` or a status code — is read as a line, not as a byte count. Unparseable input is kept
verbatim as the message, so plain text lines (no syslog header) arrive one event per line. This is how devices/MCUs
that can't run the agent feed a gateway.

**Listener hardening:** the TCP/stream-unix path bounds each frame (1 MiB) and enforces a **5-minute idle read
deadline** — a peer that stalls mid-message or opens a connection and sends nothing is closed, so an
unauthenticated slow client can't pin connections open (slow-loris). A legitimately low-volume sender that
delivers a complete message within the window is never dropped. The deadline is a fixed default, not
configurable. At most `max_connections` stream connections are served at once.

**Startup:** every listener is bound before any is served. If one cannot be bound (a port already in use, a
`socket` path that is not a socket), the input logs the error and serves nothing — no listener is left
holding its port behind the failure.

### `otlp_in` ✅ — cross-platform (OpenTelemetry logs receiver, OTLP gRPC + HTTP) — **Apex**

Receives OpenTelemetry logs over **OTLP/gRPC** and **OTLP/HTTP** (`POST /v1/logs`, protobuf or JSON, optionally
gzip) on one listener — from applications instrumented with an OpenTelemetry SDK, from OpenTelemetry Collectors'
`otlp` exporters, and from other agents' `otlp` output. The agent becomes their gateway: buffering, TLS,
reduction and any of its outputs.

| Option | Type | Default | Notes |
|---|---|---|---|
| `listen` | string | *(required)* | `host:port`, e.g. `0.0.0.0:4317`; both gRPC and HTTP are served there (cleartext HTTP/2 for gRPC without TLS, as OTLP exporters use with `insecure: true`) |
| `tls_cert_file` / `tls_key_file` | string | `""` | serve TLS |
| `tls_client_ca_file` | string | `""` | **mTLS**: require + verify client certificates against this CA |
| `auth_token` | string | `""` | when set, require `Authorization: Bearer <token>` (OTLP exporters' `headers`) |
| `max_body_bytes` | int | `4194304` (4 MiB) | largest request, decompressed — gRPC's own default message limit, which OTLP exporters respect; larger is refused (gRPC `ResourceExhausted`, HTTP `413`). A request is never refused for carrying many records or attributes: it is processed in chunks of at most 100,000 records / 1,048,576 fields, each made durable before the next, and the answer follows the last chunk (a record's attributes beyond 512 are dropped, the record is flagged `otel.attributes_truncated` and counted in `truncated_records_total`); at most 8 requests are decoded at a time, a request body must arrive within 2 minutes, and a structured value is rendered to at most 1 MiB of text |
| `commit_timeout` | duration | `60s` | longest an Export waits for its records to become durable before answering "unavailable" (the sender retries) |

**Delivery:** an Export is answered successfully only after every log record in it has left the agent's in-flight
window for good — durably spooled, delivered, or intentionally dropped — so a sender that does not get the answer
retries, and nothing it was told is accepted can be lost (at-least-once; a retry can duplicate).

**Mapping:** host = resource `host.name` (else the sender's address); source = resource `service.name` (else
`otlp`); severity from the severity number (or its text); the body becomes the message (structured bodies as JSON
text); event time from the record's time (else its observed time); every log attribute and every other resource
attribute become fields (nested values as dotted keys; a log attribute wins over a resource attribute of the same
name), plus `trace_id`, `span_id` and `otel.scope.name`. The `syslog.facility`, `syslog.procid` and `syslog.msgid`
attributes the agent's own `otlp` output writes are restored, so agent → agent over OTLP keeps every value.

### `relay_in` ✅ — cross-platform (the ack'd-transport receiver) — **Apex**

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
Loss-free by design where plain syslog/TCP is fire-and-forget. (On a RAM-only spool the ACK waits for
delivery, so the sender keeps the batch until it is delivered.)

### `beats_in` ✅ — cross-platform (Beats / Logstash protocol receiver) — **Apex**

Receives events from **Filebeat, Winlogbeat, Metricbeat and the other Elastic Beats** over the Beats protocol
(lumberjack v2 — what Beats speak to Logstash's `beats` input). Point an existing Beats fleet's
`output.logstash.hosts` at the agent and it becomes the gateway — buffering, TLS, reduction, and any of the
agent's outputs — with nothing else changed on the Beats side.

| Option | Type | Default | Notes |
|---|---|---|---|
| `listen` | string | *(required)* | `host:port`, e.g. `0.0.0.0:5044` (the Logstash Beats port) |
| `tls_cert_file` | string | `""` | serve TLS (with `tls_key_file`) — set `ssl.enabled: true` and the CA in the Beats output |
| `tls_key_file` | string | `""` | server key (with `tls_cert_file`) |
| `tls_client_ca_file` | string | `""` | **mTLS**: require + verify client certificates against this CA (Beats `ssl.certificate` / `ssl.key`) |
| `max_window` | int | `16384` | largest window (events per acknowledgement) a sender may announce; larger closes the connection. Beats send `bulk_max_size` (default 2048) |
| `max_event_bytes` | int | `4194304` (4 MiB) | largest single event; larger closes the connection |
| `keepalive` | duration | `5s` | while a window is still being processed, send a keepalive acknowledgement this often so the sender's `timeout` (default 30 s) does not fire. `0s` disables |
| `idle_timeout` | duration | `5m` | close a connection that sends nothing for this long |
| `max_fields` | int | `512` | flattened fields kept per event (in key order) |
| `keep_raw` | bool | `false` | keep the original JSON document as the event's raw payload |
| `max_connections` | int | `1024` | concurrent connections; further ones are closed at once and counted |

**Delivery:** a window is acknowledged only after every one of its events has left the agent's in-flight window
for good — durably spooled, delivered, or intentionally dropped — and windows are acknowledged in order. If the
agent stops or the connection drops before that, the Beat resends the window: at-least-once, no silent loss (a
resend after a crash can duplicate events). Compressed windows (`compression_level`, on by default in Beats) and
uncompressed ones are both accepted. A window is held in memory until it is passed on, so its total size —
decompressed — is bounded at 128 MiB, and a compressed frame inside another compressed frame is refused: a small
compressed frame cannot expand into an unbounded allocation.

**Mapping:** `message` → message; `@timestamp` → event time; host = `host.name` (then `host.hostname`,
`agent.hostname`, `agent.name`, then the sender's address); `log.level` → severity (syslog names and the Windows
level names Winlogbeat sends — `information`, `warning`, `error`, `critical`; default info); source =
`beats:<beat>` (`beats:filebeat`, `beats:winlogbeat`). Every other value becomes a field named by its dotted path
— `log.file.path`, `winlog.event_id`, `winlog.channel`, `event.code`, `tags` (arrays of plain values joined with
`,`). A document that is not a JSON object is kept verbatim as the message.

Not supported: the older lumberjack v1 protocol (logstash-forwarder), which no Beat speaks. A connection that
violates the protocol is closed and counted (`refused_connections_total` in the input's metrics, beside
`windows_total`, `events_total` and `malformed_events_total`).

### `fluent_in` ✅ — cross-platform (Fluent Bit / Fluentd Forward protocol receiver) — **Apex**

Receives logs from **Fluent Bit** and **Fluentd** over their Forward protocol (the `forward` output) — the usual
log collector in Kubernetes clusters. Point the fleet's `forward` output at the agent and it becomes the gateway:
buffering, TLS, reduction and any of the agent's outputs.

| Option | Type | Default | Notes |
|---|---|---|---|
| `listen` | string | *(required)* | `host:port`, e.g. `0.0.0.0:24224` (the Forward protocol port) |
| `shared_key` | string | `""` | require the protocol's shared-key handshake (Fluent Bit `Shared_Key`, Fluentd `<security> shared_key`); user/password authorization is not supported |
| `tls_cert_file` / `tls_key_file` | string | `""` | serve TLS (Fluent Bit `tls On`) |
| `tls_client_ca_file` | string | `""` | **mTLS**: require + verify client certificates against this CA |
| `max_message_bytes` | int | `16777216` (16 MiB) | largest request, decompressed and counting each decoded value's memory, not only its bytes on the wire (Fluent Bit chunks are typically ~2 MiB); larger closes the connection. A request also carries at most 100,000 events |
| `max_connections` | int | `1024` | concurrent connections; further ones are closed at once and counted |
| `message_keys` | []string | `[log, message, msg, MESSAGE]` | record keys tried, in order, for the event message (`log` is what Fluent Bit's `tail` and Kubernetes inputs produce) |
| `idle_timeout` | duration | `5m` | close a connection that sends nothing for this long |
| `max_fields` | int | `512` | flattened fields kept per event (in key order) |

**Delivery:** with Fluent Bit `Require_ack_response True` (Fluentd `require_ack_response true`) each chunk is
acknowledged only after every event in it has left the agent's in-flight window for good — durably spooled,
delivered, or intentionally dropped — so a crash or restart makes the sender resend instead of losing logs
(at-least-once; expect occasional duplicates after one). Without it the sender gets no acknowledgement, which is
the protocol's fire-and-forget mode. All four request forms (Message, Forward, PackedForward, gzip
CompressedPackedForward) and heartbeats are accepted.

**Mapping:** the first of `message_keys` present becomes the message (a trailing newline is trimmed; a record
with none is kept as JSON text); event time from the record's time; host from `host`, `hostname` or
`kubernetes.host`, else the sender's address; severity from `level`, `severity`, `log.level` or journald's
`PRIORITY`; source `fluent:<tag>`, and the tag as `fluent.tag` (a record cannot replace it). A host named in the
record is the sender's claim: use `shared_key` or mTLS where that matters. A record time outside years 1–9999 is
replaced by the receive time. Every other value becomes a field named by its
dotted path — `kubernetes.pod_name`, `kubernetes.namespace_name`, `kubernetes.labels.app`, `stream`.

### `gelf_in` ✅ — cross-platform (GELF receiver, UDP + TCP) — **Apex**

Receives **GELF 1.1** (Graylog Extended Log Format) — what Docker's `gelf` logging driver and many application
logging libraries send to Graylog — over UDP and TCP, so those senders can use the agent as their gateway.

| Option | Type | Default | Notes |
|---|---|---|---|
| `listen` | string | *(required)* | `host:port`, e.g. `0.0.0.0:12201` (the GELF port) |
| `protocols` | []string | `[udp, tcp]` | any of `udp`, `tcp` |
| `tls_cert_file` / `tls_key_file` | string | `""` | TLS on the TCP listener |
| `tls_client_ca_file` | string | `""` | **mTLS** on the TCP listener |
| `max_message_bytes` | int | `8388608` (8 MiB) | largest message after decompression / reassembly (Graylog's default); larger is dropped and counted |
| `udp_listeners` | int or `auto` | `1` | UDP sockets on `listen`, each read on its own core — as for `syslog_in` (1-64, `auto` = one per core, Linux and FreeBSD above 1, order across listeners not kept). Chunked messages are reassembled whichever listener each chunk reaches |
| `udp_receive_buffer_bytes` | int | `8388608` (8 MiB) | receive buffer each UDP socket asks for, as for `syslog_in`; `0` keeps the OS default |

**UDP:** one message per datagram, or chunked (at most 128 chunks, reassembled within 5 s — incomplete messages
expire and are counted), each uncompressed, gzip (the GELF default) or zlib. **TCP:** uncompressed messages ending
in a null byte (a newline is accepted too). GELF has no acknowledgement: like syslog over UDP, a sender cannot learn
whether a datagram arrived; what the agent receives it delivers at least once.

**Mapping:** `short_message` → message; `host` → host (else the sender's address); `timestamp` (epoch seconds,
fractions kept exactly) → event time; `level` → severity (GELF's default when absent is 1, alert); `full_message`,
the deprecated `facility` / `file` / `line`, and every additional `_field` (underscore removed; `_id` and boolean
values dropped, as GELF specifies) → fields; source `gelf`. The input counts `messages_total`,
`malformed_messages_total` and `expired_chunked_total`.

<a id="openwec_in"></a>
### `openwec_in` ✅ — cross-platform (OpenWEC receiver — agentless Windows hosts) — **Apex**

Receives Windows events from an [OpenWEC](https://github.com/cea-sec/openwec) collector. OpenWEC accepts
**Windows Event Forwarding** — the forwarding built into Windows, so the Windows hosts need no agent installed —
and writes each event to an output; this input is that output's receiving end. Events arrive with the same fields
as from `windows_eventlog` on the host, plus who delivered them, so detection content and host counts work the
same either way. OpenWEC is a separate program, installed and configured on its own (it handles the Kerberos or
TLS authentication of the Windows hosts and keeps their bookmarks).

| Option | Type | Default | Notes |
|---|---|---|---|
| `listen` | string | — | `host:port` for OpenWEC's `Tcp` output driver (one event per line). Set this **or** `socket` |
| `socket` | string | — | path of a Unix datagram socket for OpenWEC's `UnixDatagram` output driver (one event per datagram; not on Windows). A stale socket left at the path is replaced; any other file there is refused. The recommended transport when OpenWEC runs on the same machine — see *Choosing the transport* below |
| `socket_mode` | string | `"0600"` | octal permission bits of `socket`. Whoever can write to the socket can send events in any host's name, so only the agent's user may by default; when OpenWEC runs as its own user, use `"0660"` and a group both users share (the socket gets the agent's group, or the directory's with the set-group-ID bit). The socket never exists at its path with any other mode |
| `tls_cert_file` / `tls_key_file` | string | `""` | TLS on the `listen` socket (OpenWEC's `tls_enabled`) |
| `tls_client_ca_file` | string | `""` | **mTLS**: require OpenWEC's `tls_certificate` and verify it against this CA |
| `max_message_bytes` | int | `4194304` (4 MiB) | largest single event document; a longer line closes the connection and is counted |
| `trusted_forwarders` | []string | `[]` | principals allowed to deliver other machines' events — an intermediate Windows Event Collector that forwards on to OpenWEC. Compared case-insensitively |
| `on_identity_mismatch` | string | `flag` | `flag` forwards an event whose host is not its sender with `wef_identity: mismatch`; `drop` discards it (counted) |
| `render` | string | `message` | `message` — the rendered message the Windows host sent (subscription `content_format = "RenderedText"`), else composed from the event data; `xml` — the original event XML, as `windows_eventlog`'s `render: xml` |

**On the OpenWEC side**, give the subscription an output with `format = "RawJson"` — the original event XML
plus OpenWEC's metadata. Other formats are counted as malformed and the agent logs which format to set. Example
subscription output, for an agent listening on `127.0.0.1:5170`:

```toml
[[outputs]]
driver = "Tcp"
format = "RawJson"
config = { host = "127.0.0.1", port = 5170 }   # OpenWEC 0.3.0 names this key addr
```

or, for an agent with `socket: /run/openwec/agent.sock`:

```toml
[[outputs]]
driver = "UnixDatagram"
format = "RawJson"
config = { path = "/run/openwec/agent.sock" }
```

**Choosing the transport.** When OpenWEC runs on the same machine as the agent, use the socket. While the agent
is stopped, OpenWEC cannot write to the socket, answers the Windows host with an error, and the host sends the
events again later — in a test with the agent stopped for a minute, every event arrived once it was back. Over `Tcp`, OpenWEC 0.3.0 can lose the
events it writes just after the agent closed the connection: that write succeeds locally, OpenWEC tells the Windows
host the events were delivered, and only its next write fails. Use `Tcp` when OpenWEC runs on another machine, and
restart the agent sparingly there.

Set the subscription's `content_format = "RenderedText"` to receive the Windows host's own rendering: the message
text and the task, opcode and keyword names (`task_name`, `opcode_name`, `keyword_names`). With `Raw` the message is
composed from the event data, as for an event whose publisher has no message template. Windows gives every
forwarded Security-auditing event one and the same task name, which varies by host — `Other System Events` on a
Windows 11 host, `Process Termination` on a Windows Server 2025 domain controller, a 4624 logon included — where
an agent on the host reports `Logon`. Use `task_id`, which carries the actual task.

**Kerberos.** For Windows hosts that forward with Kerberos, OpenWEC's keytab needs entries for both the `http/`
and the `host/` principal of the collector: Windows Server 2025 asks for `host/`. The sender check below then
compares each event's host with the machine account that authenticated.

**Fields.** Every `windows_eventlog` field the event XML carries (`event_id`, `channel`, `provider`, `record_id`,
`task_id`, `keywords`, `user_sid`, `activity_id`, the event data …); host is the event's `Computer`. From OpenWEC:
`wef_source_ip` (the forwarding host's address), `wef_principal` (its Kerberos principal or TLS certificate
CommonName), `wef_subscription`, `wef_node` (when OpenWEC's `node_name` is set), `wef_time_received`, and
`wef_error` for an event OpenWEC flagged as one it could not handle (still forwarded). Account names
(`account_name` …) are not available: they would need a lookup on a domain-joined host.

**Sender check.** The `Computer` an event names is written by the forwarding host itself, so a compromised
host could send events in another machine's name. `wef_identity` records whether the authenticated sender is the
host the event names: `match`, `mismatch`, `forwarder` (a `trusted_forwarders` principal), or `unchecked` (no
principal, a user rather than machine principal, or no `Computer`). Host names are compared without their domain,
since a DNS domain can differ from the Kerberos realm, and a 15-character machine account matches the longer host
name it was truncated from.

**Through an intermediate collector.** A Windows Event Collector that passes its `ForwardedEvents` on to OpenWEC
delivers each event with the event's own channel, `record_id` and fields; only the `wef_*` fields name the
collector. Its events are `mismatch` (dropped with `on_identity_mismatch: drop`) until its principal — the
Kerberos machine account or the certificate CommonName — is in `trusted_forwarders`, after which they are
`forwarder`. A host that both forwards directly and is relayed arrives twice; the same `host`, `channel` and
`record_id` identify the duplicate.

These field names are reserved: event data with the same name arrives as `eventdata_<name>`, so an event cannot
forge them. Like syslog, OpenWEC's outputs carry no acknowledgement — what the agent receives it delivers at least
once. When one of a subscription's outputs fails, OpenWEC has the Windows host send the whole batch again, so its
other outputs can receive those events twice. The input counts `events_total`, `malformed_messages_total`, `identity_mismatch_total`,
`identity_dropped_total` and `openwec_errors_total`.

### `hec_in` ✅ — cross-platform (Splunk HTTP Event Collector compatible receiver) — **Apex**

Accepts the **Splunk HTTP Event Collector** API, so anything that can send to Splunk HEC — Docker's `splunk`
logging driver, Kubernetes Splunk connectors, products with a "send to Splunk" option, other agents' `hec` output
— can send to the agent instead, and the agent forwards wherever it is configured to.

| Option | Type | Default | Notes |
|---|---|---|---|
| `listen` | string | *(required)* | `host:port`, e.g. `0.0.0.0:8088` (the HEC port) |
| `tokens` | []string | *(required)* | the HEC token(s) senders must present (`Authorization: Splunk <token>`; `Bearer` accepted too) |
| `tls_cert_file` / `tls_key_file` | string | `""` | serve HTTPS (senders usually expect it) |
| `tls_client_ca_file` | string | `""` | **mTLS**: require + verify client certificates against this CA |
| `max_body_bytes` | int | `4194304` (4 MiB) | largest request, decompressed; larger is refused (`413`). A request carries at most 100,000 events, and one event at most 16,384 JSON keys (at any depth; more is refused with `400` "Invalid data format" before any of it is parsed); while 8 requests are already being processed a further one is answered "Server is busy" at once (senders retry) |
| `commit_timeout` | duration | `60s` | longest a request waits for its events to become durable before answering "Server is busy" (the sender retries) |

**Endpoints:** `/services/collector` and `/services/collector/event` (JSON: one object, several concatenated, or an
array), `/services/collector/raw` (one event per line; `host`, `source`, `sourcetype`, `index` from the query
string), `/services/collector/health`, and `/services/collector/ack` for senders using indexer acknowledgement —
each also under `/1.0`. gzip bodies are accepted. Errors use Splunk's codes (token required / invalid, no data,
invalid data format, event field required / blank, server busy).

**Delivery:** a request is answered `Success` only after every event in it has left the agent's in-flight window
for good — durably spooled, delivered, or intentionally dropped — so a sender that gets no answer resends
(at-least-once). A sender using indexer acknowledgement (`X-Splunk-Request-Channel`) gets an `ackId`, already
acknowledged when it polls; a channel must be a GUID-like name of at most 64 letters, digits and dashes.

**Mapping:** `event` text becomes the message; an object `event` takes its message from `message`, `line`, `log` or
`msg` (Docker's driver sends `line`) and the rest becomes fields; `time` (epoch seconds, fractions kept exactly)
→ event time; `host` → host (else the sender's address); `source` → source (else `hec`); `sourcetype` and `index`
→ fields `hec.sourcetype` / `hec.index`; `fields` → fields. The `severity` / `facility` the agent's own `hec` output
writes are restored, in either of its event formats.

### `http_in` ✅ — cross-platform (HTTP(S) ingress)

Accepts log lines over HTTP(S) `POST` — the push ingress for sources that speak HTTP but can't run the
agent or emit syslog: webhooks, IoT/edge devices, app log shippers. Each newline-delimited line of the
body becomes one event; bearer-token auth, TLS/mTLS, and a body-size cap guard the endpoint. The HTTP
analogue of `syslog_in`.

| Option | Type | Default | Notes |
|---|---|---|---|
| `listen` | string | *(required)* | `host:port`, e.g. `0.0.0.0:8080` |
| `path` | string | `/` | URL path that accepts `POST`. A path such as `/ingest` accepts only that path and other paths return 404; a path ending in `/` (the default `/` included) also accepts every path below it |
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
jittered exponential backoff (cap 30s) and re-subscribes; on shutdown it sends a graceful DISCONNECT. **QoS 2 limitation:** a QoS-2
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
| `community` | []string | `[]` (accept any, **warns**) | v1/v2c community allow-list; a mismatching community is rejected and counted. `["*"]` accepts any community on purpose, with no warning (`*` is reserved as that wildcard). An **empty list still accepts any community in this release but logs a warning at startup, and will be refused from the next minor release** — write `["*"]` or list the communities. Communities are plaintext on the wire — prefer v3 where devices support it |
| `v3_users` | list | `[]` | USM users for v3: each entry `{username, auth_protocol, auth_password, priv_protocol, priv_password, allow_legacy_crypto}`. One user entry serves any number of sending engines with that user (keys are localized per sender engine automatically) |
| `v3_users[].username` | string | *(required)* | USM security name |
| `v3_users[].auth_protocol` | string | *(required)* | `sha256` (recommended), `sha`, `sha224`, `sha384`, `sha512`, or `md5` (legacy, not FIPS-approved — see `allow_legacy_crypto`) |
| `v3_users[].auth_password` | string | *(required)* | authentication passphrase |
| `v3_users[].priv_protocol` | string | `""` (authNoPriv) | `aes` / `aes192` / `aes256` (recommended; `aes192`/`aes256` use the Blumenthal key extension that net-snmp and most devices use), `aes192c` / `aes256c` (the Reeder key extension, used by **Cisco**), or `des` (legacy, not FIPS-approved — see `allow_legacy_crypto`); empty = authenticated but unencrypted. With SHA-256 or stronger authentication the two AES-192/256 variants are identical; with SHA-1 or SHA-224 they are not, and a sender using the other variant fails to decrypt |
| `v3_users[].priv_password` | string | `""` | privacy passphrase (required when `priv_protocol` is set) |
| `v3_users[].allow_legacy_crypto` | bool | `false` | explicit opt-in for a user that relies on `auth_protocol: md5` or `priv_protocol: des`. **In this release** md5/des without the flag are still accepted (existing configurations keep working) but log a warning naming the user at startup, and **will be refused from the next minor release**; with the flag they are accepted with an info log. Either way the user is counted in the `snmptrap_legacy_crypto_users` gauge. Under `GODEBUG=fips140=only` md5, des and every SNMPv3 privacy protocol are refused at startup with an error |
| `oid_names` | map | `{}` | optional OID → name labels, e.g. `{"1.3.6.1.6.3.1.1.5.3": linkDown}` — a matching trap gains an `snmp.trap_name` field and the message text uses the names; field keys always stay numeric. No MIB compiler needed; full MIB semantics stay downstream |
| `queue_size` | int | `1024` | bounded hand-off queue between the UDP listener and the pipeline; overflow during a trap storm is counted and dropped (UDP cannot be back-pressured) |
| `source` | string | `snmptrap` | `Event.Source` label stamped on every event (the RFC 5424 APP-NAME). Before this release the default was `snmptrap_in` |
| `facility` | int | `23` (local7) | syslog facility for received traps |
| `severity` | int | `5` (notice) | syslog severity for received traps |

Each trap becomes one event: `Host` is the **UDP source address of the packet** (never the v1 trap's
agent-address field, which the device claims and a relay or a spoofed sender can set — it is kept in
`snmp.agent_address`), `Message` is the trap OID (or its `oid_names` label) plus a short varbind summary, and
the notification is carried as structured fields with the **same names and values logrok's own SNMP trap
listener stores**, so a search finds a trap whichever path delivered it (the fields are parameters of the
`logrok@66371` structured-data element, which logrok folds into `extra.snmp.*`). The community string is never
emitted.

| Field | Present | Value |
|---|---|---|
| `snmp.version` | always | `v1`, `v2c` or `v3` |
| `snmp.user` | v3 | the USM user name |
| `snmp.trap_oid` | always | notification OID with a leading dot, e.g. `.1.3.6.1.4.1.232.0.6013`; v1 traps carry the RFC 2576 translation, so a mixed v1/v2c estate is uniform |
| `snmp.enterprise_oid` | v1 | the enterprise OID, leading dot |
| `snmp.generic_trap` | v1 | net-snmp's name: `Cold Start`, `Warm Start`, `Link Down`, `Link Up`, `Authentication Failure`, `EGP Neighbor Loss`, `Enterprise Specific` |
| `snmp.specific_trap` | v1 | the specific-trap number, decimal |
| `snmp.agent_address` | v1 | the agent address the PDU states (device-claimed) |
| `snmp.uptime` | always | the device uptime as net-snmp prints it: `1:58:12.00`, or `101 days, 7:56:08.67` from one day |
| `snmp.oid.<oid>` | per varbind | one field per remaining varbind, keyed by its numeric OID **without** the leading dot (`snmp.oid.1.3.6.1.4.1.232.11.2.16.5.0`); strings as is, integers decimal, OID values with a leading dot. `oid_names` never changes the key |
| `snmp.trap_name` | `oid_names` match | the label for the trap OID |
| `snmp.inform` | informs | `true` for an acknowledged InformRequest |

The varbind keys are longer than RFC 5424's 32-byte SD-NAME limit; the syslog output keeps `snmp.oid.*` names
up to 235 bytes, so varbinds are not cut short or merged into one field. logrok's receiver accepts names up to
that length; a longer name would make it reject the whole message, so an OID beyond 235 bytes is shortened
instead. For a receiver that enforces the strict 32-byte limit, use `encoding: json`, which carries field
names unchanged. Traps are UDP:
delivery to the agent has no protocol guarantee (use informs where the device supports them — those are
acknowledged); once accepted, the pipeline's usual durability applies. Verified against the net-snmp reference
tools across the v3 auth/priv matrix (including SHA-1 and SHA-224 with AES-192/256) in the test battery.

The heartbeat/`/metrics` gauge `snmptrap_legacy_crypto_users` reports how many v3 users of the running listener
rely on md5 or des (0 when none; absent when no v3 listener runs).

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

<a id="journald"></a>
### `journald` ✅ *(Linux only)*
Reads the systemd journal through `journalctl`, which must be present on the host. On non-Linux builds this is a stub that errors if started. The JSON field
mapping is **verified against `journalctl -o json` on a live systemd host**; the `cursor_file` resume cycle
(stop → entries land while down → restart → caught up, nothing re-delivered) is **integration-tested against a
real journal** on every CI run.

| Option | Type | Default | Notes |
|---|---|---|---|
| `units` | []string | all | restrict to these units (`-u`), e.g. `[nginx.service]` |
| `identifiers` | []string | all | restrict to these `SYSLOG_IDENTIFIER` tags (`-t`, e.g. `[myapp]`) — matches `logger`/syslog(3) entries, which carry no unit |
| `priority` | string | all | max priority (`-p`): `0`–`7` or a name like `warning` |
| `cursor_file` | string | `positions/journald-<key>.cursor` beside `management.state_path` | **agent-owned** resume checkpoint: the last forwarded entry's `__CURSOR`, saved atomically (every 100 entries + on shutdown) and fed back as `--after-cursor` on restart (see [Saved read positions](#saved-read-positions)). `""` = no persistence: every start follows new entries only (`-n0`) |

Maps `PRIORITY`→severity, `SYSLOG_FACILITY`→facility (default local0/16), `_HOSTNAME`→host,
`_SYSTEMD_UNIT`/`SYSLOG_IDENTIFIER`→`journald:<unit>` source, `__REALTIME_TIMESTAMP`→timestamp, plus
`unit`/`pid`/`uid`/`syslog_identifier`/`boot_id` fields (`uid` = the journal's kernel-verified `_UID` sender —
the user-traceability field compliance mappings reference). Resume via `cursor_file` is bounded-dup at the crash boundary
(checkpoint trails the forwarded send — a crash replays the unsaved tail, never skips it; no entry ack — see
the at-least-once note under [`buffer`](#buffer)). The agent saves this resume position itself, so a resume is never lossy, even after
the service is killed.

**Future-dated journals.** `journalctl --follow` waits by time, so a journal whose newest entry is dated in the future (a host after an RTC or far-future clock fault) can leave a follow that never delivers new, correctly-dated entries. The input checks the journal's newest entry at start and on every forwarded entry; if it is more than 5 minutes ahead of the clock it logs a WARN and switches to *cursor polling* — a non-follow `journalctl --after-cursor=<last>` about every 2 seconds, which is position-anchored. Delivery stays at-least-once with the same `cursor_file` semantics; the cost is up to ~2 s extra latency and one process spawn per cycle. Healthy journals keep the single long-lived follow. No option; restart the agent to return to follow once the bad entries are no longer the newest.

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
| `read_from` | string | `end` | file mode: `end` (new records only) or `beginning` (whole file) — applies only when there is no saved position. Invalid in multicast mode |
| `state_path` | string | file mode: `positions/linux_audit-<key>.json` beside `management.state_path`; multicast: none | file mode: where the read position is saved (see [Saved read positions](#saved-read-positions) and *Checkpointing* below). `""` = no persistence (every restart re-applies `read_from`). Refused in multicast mode, which cannot resume |
| `reassembly_timeout` | duration | `2s` | complete an audit event whose `EOE` terminator never arrives (EOE isn't guaranteed in file mode): once a record this much newer (by its audit timestamp) has been read, or — with the log quiet — after this long on the clock. Records that always stand alone (user-space messages such as `USER_LOGIN`, `CRED_ACQ`, `SERVICE_START`, daemon and anomaly messages) complete their event at once and never wait |
| `max_in_flight` | int | `1024` | bound on concurrently-buffered audit events (serials) during reassembly; overflow emits the oldest event as-is with a counted warning — memory never grows unbounded |
| `source` | string | `linux_audit` | `Event.Source` label |

**Mode tradeoff.** `file` works on every distro running auditd and needs only read permission on
`audit.log` (root or the `adm`/`root` group, distro-dependent) — use it by default. `multicast` reads the
kernel's read-only live audit feed (kernel ≥ 3.16): a live kernel feed with no file I/O and no dependency on auditd's log format/rotation settings — but it requires
**`CAP_AUDIT_READ`** (typically root; without it the input fails at start with an explicit "requires
CAP_AUDIT_READ … or use mode: file" error), and records the kernel emitted while the agent was down are
missed (the file has them). Multicast is **strictly passive**: the agent never registers as the audit
daemon, so a running auditd keeps working, and keeps writing `audit.log`,
untouched. Neither mode loads audit *rules* — configure those with `auditctl`/`augenrules` as usual; with
no rules loaded there is simply no traffic.

**Checkpointing.** File mode saves its read position (a byte offset plus a fingerprint of the log's
first line) to `state_path`, by default, and a restart resumes there — records written while the agent
was stopped are delivered, not skipped. The saved offset only moves past an audit event once the
pipeline has durably handled it (delivered, spooled or intentionally dropped), and never past a record
still being reassembled, so a crash re-sends at worst a short tail (at-least-once; records of
interleaved events can repeat), never loses one. The position is saved during a long catch-up, not only
at its end. If `audit.log` was rotated while the agent was stopped, the rest of the rotated file
(`audit.log.1`) is read before the new one; a log that was rewritten or replaced is read from its start.
`read_from` applies only to a start with no saved position. **Multicast mode cannot resume**: the
kernel stream cannot be replayed, so records emitted while the agent is down are missed (the file has
them).

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


### `bsm_audit` ✅ *(Solaris and other BSM systems)* — **Apex**

Collects the **Solaris BSM audit trail** — logins, privilege use, process execution and file changes — the
NXLog `im_bsm` counterpart, and the file-integrity evidence source on the extended-platform tier. Each record
arrives with the **actor**: the audit user, real user, process and session that caused it.

**It decodes the trail with the operating system's own `praudit`**, which must be present on the host.

**Fields are mapped generically.** Every element of a record becomes fields — `<subject>` attributes as
`subject_uid`, `subject_audit-uid`, `subject_pid`, a `<path>` as `path`, a second one as `path.2`, and so on.
Nothing enumerates token types, so a record type this release has never seen still arrives intact. The
repeat-indexing convention is the same one `linux_audit` uses, so both trails read the same way.

**Two prerequisites, and both are yours rather than the agent's:**

1. **Read access to the trail.** `/var/audit` is `root:root` mode `0640`. The agent must be able to read it.
2. **The audit flags select what is collected.** File-change records need the file classes enabled
   (`auditconfig`), exactly as `linux_audit` needs auditd rules. **With no relevant flags there is simply no
   traffic** — which looks identical to a broken input, so check the flags before suspecting the agent.

| Option | Type | Default | Notes |
|---|---|---|---|
| `dir` | string | `/var/audit` | Directory auditd writes trails to. The **active** trail is the one named `*.not_terminated.*`; on rotation auditd renames it and creates a new one, and the input follows across |
| `read_from` | string | `end` | `end` (new records only) or `beginning` (the whole active trail). **There are only two** because praudit cannot be started at an arbitrary offset — see the checkpoint note below |
| `praudit` | string | `praudit` | Path to the decoder, if it is not on `PATH` |
| `poll_interval` | duration | `1s` | How often to check for new bytes and for a rotated trail |
| `source` | string | `bsm_audit` | `Event.Source` label |

Severity is notice (5) and facility 13 (`log audit`) — the same pair `linux_audit` uses, so both trails land
identically in a receiver's routing rules.

**Checkpointing (documented limitation).** Unlike `linux_audit`'s file mode, this input persists **no byte offset**: a
restart resumes per `read_from`. This is a consequence of the format, not an omission — `praudit` fails on a
stream that begins mid-record (`No code associated with token id …`), because BSM framing is only recoverable
from a record boundary, and finding one would mean parsing the binary layout the subprocess exists to avoid.
Only offset 0 and end-of-file are guaranteed boundaries. Records missed during a restart remain in the trail
file and can be replayed with `read_from: beginning`.

```yaml
inputs:
  - type: bsm_audit
    # dir: /var/audit
    # read_from: end          # end (default) | beginning
    # poll_interval: 1s
```
<a id="oslog"></a>
### `oslog` ✅ *(macOS only)* — **Apex**
Reads the macOS unified log (OSLog) through `/usr/bin/log`. On non-macOS builds this is a stub that
errors if started. On startup it **backfills** the gap since the last run via `log show --start <checkpoint>`,
then follows the live log via `log stream`. The `log` binary is always invoked by absolute path
(`/usr/bin/log`) because an interactive shell has a `log` builtin that would otherwise shadow it.

| Option | Type | Default | Notes |
|---|---|---|---|
| `predicate` | string | `""` | optional NSPredicate passed verbatim to `--predicate`, e.g. `subsystem == "com.apple.securityd"` |
| `level` | string | `default` | `default` \| `info` \| `debug` — widen captured verbosity (`--level` for stream, `--info`/`--debug` for the backfill) |
| `checkpoint_file` | string | `positions/oslog-<key>.checkpoint` beside `management.state_path` | **agent-owned** resume checkpoint: the last forwarded event's `timestamp`, saved atomically (every 100 events + on shutdown) and fed back as `log show --start` on restart (see [Saved read positions](#saved-read-positions)). `""` = no persistence: live stream only, no backfill |
| `backfill_on_start` | bool | `true` | when a checkpoint exists, `log show` the gap before streaming; set `false` to stream live only |

Maps `messageType`→severity (`Fault`→crit/2, `Error`→err/3, `Default`→notice/5, `Info`→info/6, `Debug`→debug/7),
`eventMessage`→message, `processImagePath` basename→`oslog:<process>` source, `timestamp`→timestamp, plus
`process`/`process_image_path`/`pid`/`subsystem`/`category`/`sender_image_path`/`thread_id`/`trace_id`/`activity_id`/`message_type`/`format_string`
fields (`format_string` is the un-interpolated OSLog message template — the natural key for
event-template grouping/dedup; `pid` also rides in RFC 5424 PROCID, see [`syslog` output](#syslog---rfc-5424-over-tcp-tlsmtls-or-udp-diode-mode)). The unified log carries no hostname, so `host` is the agent's `os.Hostname()`. Resume via
`checkpoint_file` is bounded-dup at the boundary (`--start` is inclusive; the checkpoint trails the forwarded
send — a crash replays the unsaved tail, never skips it; same at-least-once class as journald).

---

<a id="processors"></a>
## `processors`

Run in order; a processor returning *drop* removes the event.

### `add_fields` ✅

Stamps static key/value fields onto every event — tenant, site, environment, device type. Use it to
label a fleet at the edge so events stay attributable downstream without receiver-side lookups; the
added fields travel like any other event field (RFC 5424 structured data on the `syslog` output).

| Option | Type | Notes |
|---|---|---|
| `fields` | map | static key/value fields added to every event (values stringified) |

**Agent identity (always on, no configuration).** Before any configured processor, every event the agent
collected itself gets `agent_type: luna`, `agent_version` (the binary's version) and, once the agent is
enrolled, `agent_id`. Events a receiver input took from another collector (`syslog_in`, `otlp_in`,
`beats_in`, `fluent_in`, `hec_in`, `gelf_in`, `http_in`, `openwec_in`) are not stamped, and an event that
already carries `agent_type` (an edge agent's, through `relay_in` or `syslog_in`) keeps it. Windows event
data named like these fields is stored as `eventdata_<name>`.

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
| `drop_message` | bool | `false` | blank the message after a **successful** parse (failed parses keep it; only when `source` is the message) |
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
capped (32 levels).

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
    - if: 'severity >= 5'            # drop notice/info/debug noise (syslog: higher = less severe)
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
When `suppress_count` is enabled (default on; it exists on `throttle`, `dedup` and `quota` — `sample` has no such option), the next event that passes after a suppressed run carries a
`suppress_count` field with the number of events that were dropped. `quota` reports the gap differently: the
first event admitted after budget-drops carries `quota_dropped` and `quota_dropped_bytes` instead.

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
| `keep_if` | string | `""` | `expr`-grammar condition; matching events bypass the budget |

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

#### `aggregate` ✅ — per-group counts per time window — **Apex**

Replaces the events of each group in each fixed window with **one summary event** carrying the count — so a
flood of repetitive events (firewall accepts, load-balancer health checks, routine audit noise) reaches the
SIEM as a handful of numbers instead of thousands of lines. Windows are aligned to the clock (a 1-minute
window runs from :00 to :00). At the end of each window the agent emits, per group:

- `Message`: `<count> events in <window>` followed by the group values (`host=fw1 action=accept`);
- fields: each `group_by` field with its value, `aggregate_count`, `aggregate_window_start` and
  `aggregate_window_end` (RFC 3339, UTC), and `aggregate_sum_<field>` for each `sum_fields` entry that held
  numbers in that window;
- `Timestamp` = the window end, `Severity` = the most severe severity counted, `Source` =
  `aggregate:<name>`, `Host` = the group's host (when `host` is in `group_by`).

Summaries pass through the processors after `aggregate` and go to every destination like any event. They
also get the tags that `add_fields` processors earlier in the chain add, and the agent group on a managed
agent, so a summary carries the same identity as the events it replaces; earlier parsers and filters are not
re-run on it. The
originals are dropped (counted in `processor_<name>_dropped_total`) unless `keep_originals: true`. **A
crash or hard kill loses the summaries of windows still open** — at most one window per group — because the
originals' checkpoints were already committed; a clean stop (or reload) emits them. Use `keep_originals`
where every event must be kept and the summary is extra.

| Option | Type | Default | Notes |
|---|---|---|---|
| `group_by` | []string | `[host, source]` | fields that form a group: `host`, `source`, `severity`, `facility` or any event field. `message` and `*` are refused (every distinct event would be its own group — use `dedup`) |
| `window` | duration | `1m` | window length, at least `1s` |
| `max_groups` | int | `10000` | open groups at once; events of a new group past the cap **pass through un-aggregated** (never dropped). The first time the cap is hit the agent logs one WARN naming the processor and the cap, and every such event is counted in `processor_<name>_overflow_total` (`logrok_agent_processor_overflow_total{processor="<name>"}` on `/metrics`, also in the heartbeat). A non-zero count means the reduction is not applying to some traffic: raise `max_groups` or narrow `group_by` |
| `sum_fields` | []string | `[]` | numeric fields to add up per group (e.g. `bytes`); non-numeric values are ignored |
| `keep_originals` | bool | `false` | also pass every original event on |
| `keep_min_severity` | int | *(none)* | events with `Severity <= N` pass untouched and are not counted |
| `keep_if` | string | `""` | `expr`-grammar condition; matching events pass untouched and are not counted |

```yaml
- type: aggregate
  group_by: [host, action, dst_port]
  window: 1m
  sum_fields: [bytes]
  keep_min_severity: 3        # errors and worse still arrive one by one
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
| `mode` | string | `mask` | `mask` (replace), `hash` (replaced by `[REDACTED:<first 16 hex characters of the salted SHA-256 digest>]`), or `remove` (delete) |
| `replacement` | string | `[REDACTED]` | Replacement text in `mask` mode |
| `hash_salt` | string | `""` | Salt for `hash` mode. Agents sharing a salt produce equal digests for equal inputs, which is what makes correlation possible without retaining the value |
| `fields` | []string | `[]` | Restrict redaction to these field names. Empty = every field and the message. When set, the message is redacted only if `message` is in the list |
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
  file: /etc/logrok-universal-agent/assets.csv
  format: csv                      # host,owner,env,site
  fields: [owner, env]             # ignore the other inventory columns
  prefix: asset_                   # -> asset_owner, asset_env
  reload: 5m                       # pick up inventory changes without a restart
  case_insensitive: true
  on_missing: default
  default: { owner: unassigned, env: unknown }
```

#### `file_hash` ✅ — attach a changed file's digest to the event that reported the change — **Apex**

Reads a path out of the event, hashes that file, and adds the digest to the event. Never drops events.

**This is the content half of file-integrity evidence.** The agent's file-change events already carry the
**actor** — who touched the path, when, with which command line (auditd watch rules on Linux; Sysmon or
object-access auditing on Windows). What they don't carry is *what the file now is*. Adding the digest here
lets your platform compare content against whatever baseline it keeps, while the event still names the person
responsible. The agent keeps no baseline and makes no judgement about whether a change was authorised — see
[File-integrity events](USER-GUIDE.md#file-integrity-events-fim--what-the-agent-does-and-what-it-deliberately-doesnt).

**`roots` is required, and it is a security control.** The path comes out of an event, and on a pipeline fed
by `http_in` or a relay an event can be attacker-influenced. Without a bound, "hash the path in this field"
would let a crafted event ask the agent to read any file its user can open and publish a digest of it — which
for a small or known-format file is a disclosure, not a checksum. Symlinks are resolved **before** the check,
so a link inside an allowed directory cannot reach outside it. There is deliberately no permissive default.

**The digest describes the file when it was hashed, not when it changed.** The event arrives after the write,
so the file may have been written again, replaced or deleted in between. That race is inherent to hashing an
event stream. Every event that is looked at therefore gets a `<prefix>hash_status` saying what happened (an event that already carries a digest is passed through untouched, without a status, unless `overwrite: true`), so a missing digest is
a recorded outcome rather than a silently absent field.

| Option | Type | Default | Notes |
|---|---|---|---|
| `path_fields` | list | *(required)* | Event fields that may hold the path; the **first non-empty one wins**. One processor serves several sources: `[path, TargetFilename, ObjectName]` covers auditd, Sysmon and Windows object-access auditing |
| `roots` | list | *(required)* | Directories whose files may be hashed, e.g. `[/etc, /usr/bin]`. A path resolving outside every root is refused with status `denied`. **No default — see above** |
| `algorithm` | string | `sha256` | `sha256`, `sha512` or `sha1`. `sha1` is offered so you can compare against an existing scan-and-checksum FIM baseline without rehashing an estate; it is not the default |
| `max_size` | int | `67108864` (64 MiB) | Files larger than this are skipped with status `too_large`. Hashing is O(size) on the pipeline, so an unbounded cap turns one large file landing in a watched directory into a collection stall |
| `prefix` | string | `file_` | Prefix for the added fields → `file_hash`, `file_hash_algo`, `file_size`, `file_hash_status` |
| `cache` | int | `1024` | Remembered `(path, mtime, size)` → digest entries. A file under active write produces many events; without the memo each costs a full re-read |
| `overwrite` | bool | `false` | Re-hash even when the event already carries a digest. Leave off behind a relay: the upstream hop hashed the file where it actually lives, and this host's filesystem is the wrong one to consult |

`<prefix>hash_status` is set on every event the processor hashes or tries to hash, to exactly one of:

| Status | Meaning |
|---|---|
| `ok` | hashed; `file_hash`, `file_hash_algo` and `file_size` are set |
| `no_path` | none of `path_fields` held a value — usually a `path_fields` list that doesn't match this source |
| `not_found` | the file was gone by the time it was read (the race above) |
| `denied` | resolved outside every configured root, or could not be resolved at all |
| `too_large` | above `max_size` |
| `not_regular` | a directory, device, socket or FIFO. Not merely uninteresting — hashing a device node never returns |
| `unreadable` | permissions or an I/O error |

```yaml
- type: file_hash
  path_fields: [path, TargetFilename, ObjectName]   # auditd / Sysmon / object-access auditing
  roots: [/etc, /usr/bin, /usr/sbin]                # required: only these may be hashed
  algorithm: sha256
  max_size: 33554432                                # 32 MiB
  prefix: file_                                     # -> file_hash, file_hash_status, ...
```

#### `noise_fingerprint` ✅ — enforce logrok noise decisions at the sender — **Apex**

logrok classifies log templates as noise and gives each template a fingerprint (the template
the operator sees in logrok). This processor computes the same
fingerprint of every event's message — and the legacy fingerprint older platform versions send, so either
matches — and, when one is in the configured set, drops the event or marks it —
the cost per event does not grow with the set, so thousands of templates are fine. A managed or hand-written config carries the
set (logrok filling it in automatically follows on its side). By default events of severity err and worse
always pass (the same line logrok draws at ingest). **Place it first in the processors list**: it hashes
the message as received, and anything that rewrites the message (`redact`, `trim_fields`, the parsers)
would change the fingerprint. **It cannot match on a `snare` destination or a `syslog` destination with
`encoding: json`** — those rewrite the message before it is sent, so logrok fingerprints different bytes;
the agent warns at startup when it sees that combination. A message truncated on the way (UDP datagram
size, a receiver's message-size cap) cannot match either.

| Option | Type | Default | Notes |
|---|---|---|---|
| `fingerprints` | list or map | `[]` | sha256 hex fingerprints — a list of strings, or a map `fingerprint: template id` (what logrok generates; the id is what a marked event carries). Lowercase hex only; an empty set is a no-op |
| `action` | `drop` \| `tag` \| `route` | `drop` | `drop` removes the event on the host; `tag` and `route` both keep it and set `logrok_noise=<action>` and `logrok_noise_template=<id>` (the fingerprint when no id is known), which ride into logrok as structured data. On the agent the two are identical — the value only tells logrok which decision to apply at ingest |
| `keep_min_severity` | int | `3` | Events with `Severity <= N` are neither dropped nor marked (default: err and worse) |
| `keep_if` | string | `""` | `expr`-grammar condition; matching events pass untouched |

```yaml
- type: noise_fingerprint
  name: logrok-noise-drop        # the processor label: logrok_agent_processor_dropped_total{processor="logrok-noise-drop"}
  action: drop                   # default
  fingerprints:
    262af52f290b7ce1b309f5135fb2d9a1af63b6117d347f872595b024463b064a: tmpl-7f3a   # "wakeup dt=<*>"
    9038a40dd133bf7037606098aea90b257ed2bcb8d8b8138cc735e96d14490bbe: tmpl-c1d2   # "state update"
  keep_min_severity: 3           # default
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
  combined with `&&`/`||`/`!`) restricts which events reach that destination. Absent = the
  destination receives every event. An event matching **no** destination is intentionally
  discarded (counted under the `routing_when` drop family) — **a `when:`-filtered event is gone; it
  is not spooled anywhere**, so keep a catch-all destination if you need one. A bad expression is
  rejected when the configuration is applied (a pushed config rolls back). Routing requires the
  same Apex entitlement as fan-out; a degraded agent clears `when:` conditions with a loud warning
  so it never silently drops events you believe are being collected.
- **Per-output `buffer:` override** — `max_bytes` and `when_full` may be set per output (a lossy
  best-effort sink and a must-not-drop SIEM feed legitimately want different policies). Anything not
  overridden inherits the global `buffer` settings. An explicit `max_bytes: 0` means unlimited.
- **Per-output `max_in_flight`** — how many batches the agent may have on the way to this output at
  once. Default `1`: one batch at a time, each sent only after the previous one was answered. Range `1`
  to `64`. Accepted only by outputs that send each batch as its own request and answer — today the
  `hec` output; on any other output the configuration check refuses the key (and an agent starting
  with it logs a warning and sends one batch at a time). See [`hec`](#hec-max-in-flight) for when to
  raise it. What stays the same: an input's position still advances strictly in the order events were
  read, a batch that fails is spooled before anything read after it is acknowledged to its input, and
  delivery stays at-least-once. What changes: **the receiver may get batches in a different order**
  than they were read (events inside one batch keep their order), and the agent holds up to that many
  batches in memory for the output while they are on the way. Once a batch fails, the agent stops
  sending ahead until the failed batch is spooled; new events then queue behind it. At shutdown the
  agent waits for batches on the way up to `service.shutdown_grace`; a batch still unanswered then is
  spooled if a disk spool is configured, and otherwise left unacknowledged so the input reads it again
  after the restart.
- **Delivery semantics.** An input's checkpoint advances only when **every** destination holds the
  event durably (delivered or on its spool) — so no destination can silently miss events the agent
  has acknowledged to its source. Under `when_full: block`, a full destination back-pressures the
  whole pipeline (the slowest destination governs, by design — `block` means "never lose").
- **Use a disk spool with routing.** With `when:` routing, an input saves its read position only past events
  that every output has finished with. While one output is down with only the in-memory spool, events that
  went to other outputs (or matched no route there) wait behind the events that output still holds in memory,
  so a restart re-reads them all. At most 65,536 such waiting events are kept per output; what happens past
  that follows the output's `when_full`:
  - `when_full: block` — the agent slows intake until the output drains its backlog instead of moving the
    input's position early: nothing is lost, and `output_<name>_backpressure_waits_total` climbs while it
    waits. If the agent stops during the wait, those events are read again after the restart.
  - `drop_oldest` (the default) or `drop_newest` — the input's position moves on early (the agent logs a
    warning) and each such event is counted in `output_<name>_held_commit_overflow_total`; a crash or restart
    can then lose events that were still only in that output's memory.

  A disk spool (`buffer.enabled: true`) avoids both: queued events are on disk, so nothing has to wait.
- **Spool layout & scaling.** With one output the spool lives directly in `buffer.dir` (unchanged
  from earlier releases). With several, each destination gets `buffer.dir/<name>`. The agent
  migrates automatically in both directions: going from one output to several moves an existing
  root-layout backlog into the *first* output's subdirectory; going back to one output moves that
  output's subdirectory back. A backlog is never silently stranded — subdirectories that match no
  configured output name are reported loudly at startup ("orphaned spool subdirectories"), and a
  migration that would overwrite existing spool files refuses and fails the config load instead.
- **Metrics.** Each destination exports its own series —
  `logrok_agent_output_events_out_total{output="siem"}`, `..._events_dropped_total`,
  `..._buffer_depth`, `..._spool_generation`, `..._backpressure_waits_total`,
  `..._held_commit_overflow_total`, `..._oldest_unsent_age_seconds` — while the fleet-wide unlabeled totals remain the sums
  across destinations (the oldest-unsent age: the largest).

<a id="syslog-output"></a>
### `syslog` ✅ — RFC 5424 over TCP (TLS/mTLS) or UDP (diode mode)

The default output: forwards every event as **RFC 5424 syslog** — the vendor-neutral wire format any
aggregator, collector, or SIEM accepts (a logrok deployment's syslog-ng front door by default, but
equally any third-party receiver). TCP with TLS/mTLS is the normal production shape; UDP exists for
hardware data diodes and plain `udp()` receivers. Structured fields survive the trip as RFC 5424
structured data — or as a JSON message body with `encoding: json`.

| Option | Type | Default | Notes |
|---|---|---|---|
| `endpoint` | string | *(required)* | `host:port` of the aggregator (logrok syslog-ng by default) |
| `protocol` | string | `tcp` | `tcp` or `udp`. UDP sends **one datagram per message** (RFC 5426, no framing) — required for **hardware data diodes** (Waterfall/Owl pass UDP only) and plain `udp()` receivers. UDP is fire-and-forget: no delivery guarantee on the wire; pair with `sequence` for receiver-side gap detection **UDP diode mode is Apex**; RFC 5424 over TCP/TLS is Core. |
| `failover_endpoints` | list of `host:port` | *(none)* | **Core** (free on `syslog`; on `snare` it follows that output's Apex tier). Tried **in order** when the active endpoint fails to connect or write — inside the same delivery attempt, so the spool is the last resort, not the first. The primary `endpoint` stays first. Entries must be unique and non-empty; not supported with `protocol: udp` (no failure signal). TLS settings apply to every endpoint; with `server_name` set, every endpoint must present that name. A failed write mid-batch resends the whole batch on the next endpoint (the same bounded-duplicate class as a re-spool). A failback happens between batches, so the first batch after a failback is paced; the first batch after a mid-write failover is sent unpaced and the drain-pacing ramp restarts on the next batch. |
| `failback_every` | duration | `60s` (when `failover_endpoints` is set) | While on a failover endpoint, probe the primary this often and return to it on success — between batches, never inside one. `0s` = stay on the secondary until it fails. Ignored without `failover_endpoints`. Worst case, the probe adds up to `min(output timeout, 2s)` to one batch per interval while the primary is black-holed (dial doesn't return promptly). |
| `framing` | string | `newline` | **TCP only.** `newline` (one message per LF — what syslog-ng's `network()` source and most receivers expect) or `octet-counting` (RFC 6587 `LEN SP MSG`; preserves embedded newlines but needs an octet-counting-aware receiver, e.g. syslog-ng's `syslog()` source). Setting it with `protocol: udp` is a config error. **Interop:** syslog-ng `syslog()` sources, including appliance RFC 5424 ports (601) and RFC 5425/6587 listeners, expect `octet-counting` — they accept a `newline`-framed connection without any error on either side and deliver nothing. Use `newline` for `network()`-style receivers. When an endpoint's port is 601 or 6587 with `newline` framing, the agent logs a WARN at startup and `-check-config` lists a warning |
| `header_from_fields` | bool | `true` | Build the RFC 5424 header from the event's own header fields: **APP-NAME** from the `app` field, **PROCID** from `procid` (else `pid`), **MSGID** from `msgid` — set by `syslog_in` from a received message's header (and from an RFC 3164 tag with `parse_tag`), and by any input's `add_fields` (e.g. `add_fields: {app: webapp}` on a `filetail` input). Each falls back as described below when the field is absent or not valid for its header slot. `false` restores the previous header exactly — APP-NAME = the event source (`syslog-in` for everything a gateway forwards), PROCID from `pid` only, MSGID `-` — for receivers that key on that APP-NAME. The fields still travel as structured data (or in the JSON body) either way |
| `sequence` | bool | `false` | add a `[seq@66371 session=".." n=".."]` SD element with a per-event monotonic counter. A receiver behind a one-way link detects **loss** (gaps in `n`) and **agent restarts** (`session` change). Counters are assigned at send time, so retried batches get fresh numbers — `n` detects gaps, not duplicates |
| `max_datagram_size` | int | `8192` | **UDP only.** Datagrams are truncated to this many bytes (an over-MTU/oversized send would otherwise fail forever and wedge the spool drain). A truncation counts `logrok_agent_udp_truncated_total` and logs a one-time WARN, so a too-small value is visible rather than silently cutting messages — raise it if your receiver accepts more |
| `encoding` | string | `text` | `text` (message verbatim, structured fields as a `[logrok@66371 ...]` SD element) or `json` (the RFC 5424 envelope stays, but MSG is **one JSON object** — `timestamp`, `host`, `source`, `severity`, `facility`, `message`, `fields`, plus `tz_offset` when the source's timestamp carried a non-zero UTC offset — and the fields SD element is dropped, no duplication). Use `json` for receivers that parse JSON natively (syslog-ng `json-parser()`, Splunk, Logstash, Loki). Bonus: JSON escapes control characters, so **multi-line messages survive newline framing** as a single frame — no `octet-counting` receiver needed. The `sequence` SD element still applies. ⚠️ **Keep `text` when forwarding into logrok**: its field extraction reads the `[logrok@66371]` SD element, which `json` drops — `json` would land fields (incl. `agent_group`) unparsed inside the message |
| `tls` | bool | `false` | enable TLS (**TCP only** — no DTLS; `protocol: udp` + `tls` is a config error) |
| `ca_file` | string | system roots | CA bundle to **verify the server** (RootCAs) |
| `cert_file` | string | `""` | client cert for **mTLS** (requires `key_file`); PEM holding the full chain — the certificate, then any intermediates (with only the leaf, a receiver that lists its root CA is not offered a certificate at all). It should carry the `clientAuth` extended key usage or none at all: a receiver that checks key usage (logrok's agent listener does) refuses one limited to e.g. `serverAuth` — the agent warns at startup and `-check-config` lists it under `warnings` |
| `key_file` | string | `""` | client key for mTLS (requires `cert_file`) |
| `server_name` | string | from `endpoint` | verification name / SNI override |
| `cert_source` | string | `static` | `static` (use `cert_file`/`key_file`) or `enrolled` (present the control-plane-issued cert from `management.tls.mode: enrolled`; mutually exclusive with `cert_file`/`key_file`). The agent *presents* the cert; the receiver must verify it (`peer-verify`/`ca-file`) for enforcement |
| `insecure_skip_verify` | bool | `false` | **DEV ONLY** — disables server verification |
| `drain_pacing` | block | *(off)* | **Reconnect drain pacing** for this unacked transport. Omit → the spool drains at line rate (default). Set it when a high-volume plain-syslog feed can overrun a bounded receiver on reconnect: after an outage the durable spool would otherwise re-blast the whole backlog at once, and because plain syslog has **no delivery ack** the receiver's overflow drops are invisible to the agent (it can silently lose its own backlog). Pacing caps the drain rate and eases a recovered receiver back in. **Only for `syslog`/`snare`** (unacked); the acked outputs (`otlp`/`relay`, HEC-ack) are paced by their receiver's acknowledgment and ignore this. Sub-keys below |
| `drain_pacing.rate_eps` | int | *(required in block)* | Steady events/sec cap while **draining a backlog** (0/omitted = unlimited, i.e. the block does nothing; `slow_start`, `ramp_start_eps` or `jitter` set with it is a configuration error). Steady-state forwarding at or below this rate is never throttled |
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
`procid` field (a message received by `syslog_in`) or a `pid` field (Windows Event Log, macOS `oslog`,
journald), it is emitted in the RFC 5424 **PROCID** header field — `procid` first, so a forwarded message keeps
its sender's PROCID — so a collector can index the process id as a first-class column rather than parse it out of
structured data (it also stays in SD). The value must be 1–128 printable ASCII characters without spaces; events
without a usable one emit PROCID `-` (NILVALUE). A `msgid` field (a received message's MSGID) is emitted as the
**MSGID** header field when it is 1–32 printable ASCII characters without spaces, else `-`.
The **APP-NAME** header field is the event's `app` field when it has one (the program a `syslog_in` sender named,
or one set with `add_fields`), otherwise the event source (`file:<path>`, `journald:<unit>`, `oslog:<process>`,
`syslog-in`, …). Either is made grammar-valid on the way out: characters outside printable ASCII (incl.
spaces) become `_` and the value is capped at the RFC's 48-character maximum — a raw source like
`file:/var/log/my app.log` would otherwise shift every later header field on a spec-following receiver.
Lossless: whenever APP-NAME is not exactly the event source — because sanitization changed it, or because the
`app` field replaced it — the full source is added as a `source="…"` SD param (`encoding: json` always carries it
in the JSON body) — so APP-NAME is the indexable shorthand, never the only copy, and a gateway's events still say
they came through `syslog-in`. A user field already named `source` takes precedence. The `app`, `procid` and
`msgid` fields also stay in structured data, where receivers that read them today keep finding them.
`header_from_fields: false` turns the `app`/`procid`/`msgid` header rules off (see the table).
The **HOSTNAME** header field gets the same treatment (it is peer-controllable via the `syslogin`/`relay_in`
receivers): bytes outside printable ASCII — including spaces, which would otherwise shift every later header
field — become `_`, the value is capped at the RFC's 255-byte maximum, and the full original rides as a
`host="…"` SD param when sanitization changed it (`encoding: json` carries it in the body). Structured-data
**field keys** (SD-NAME) — arbitrary via `parse_json`/`parse_kv`/`parse_csv` and `relay_in` — are likewise
made grammar-valid: out-of-charset bytes become `_`, keys cap at the RFC's 32-character maximum (except `snmp.oid.*` keys from `snmptrap_in`, kept up to 235 bytes), and an empty
key becomes `_`. The **PRI** value is clamped defensively — severity into 0–7, facility into 0–23 — so a
peer-injected out-of-range value can never emit a malformed `<247>`/`<-1>` frame a strict receiver rejects.
Server verification is **on by default** when `tls: true`; `insecure_skip_verify` is the only way to disable it.

**Metrics.** Per destination: `output_<name>_failovers_total`, `output_<name>_failbacks_total`,
`output_<name>_active_endpoint` (0 = primary, 1+ = position in `failover_endpoints`); on `/metrics` the
labelled families `logrok_agent_output_failovers_total`, `…_failbacks_total`, `…_active_endpoint`. Present
only for destinations that configure failover.

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
| `framing` | string | `newline` | **TCP only**, same as `syslog`: `newline` (one line per LF — the normal Snare-over-syslog wire) or `octet-counting` (RFC 6587). A Snare record is single-line by construction (embedded tabs/newlines in a field are neutralized to spaces so they can't shift columns). The port 601/6587 `newline` framing warning of the `syslog` output applies here too |
| `max_datagram_size` | int | `8192` | **UDP only**, same as `syslog` |
| `tls` | bool | `false` | enable TLS (**TCP only**) — identical surface to the `syslog` output |
| `ca_file` | string | system roots | CA bundle to **verify the server** (RootCAs) |
| `cert_file` | string | `""` | client cert for **mTLS** (requires `key_file`); PEM holding the full chain — the certificate, then any intermediates (with only the leaf, a receiver that lists its root CA is not offered a certificate at all). It should carry the `clientAuth` extended key usage or none at all: a receiver that checks key usage (logrok's agent listener does) refuses one limited to e.g. `serverAuth` — the agent warns at startup and `-check-config` lists it under `warnings` |
| `key_file` | string | `""` | client key for mTLS (requires `cert_file`) |
| `server_name` | string | from `endpoint` | verification name / SNI override |
| `cert_source` | string | `static` | `static` (use `cert_file`/`key_file`) or `enrolled` (present the control-plane-issued cert from `management.tls.mode: enrolled`; mutually exclusive with `cert_file`/`key_file`) |
| `insecure_skip_verify` | bool | `false` | **DEV ONLY** — disables server verification |

`failover_endpoints` and `failback_every` (see the
[`syslog` output](#syslog---rfc-5424-over-tcp-tlsmtls-or-udp-diode-mode)) work identically here — one primary,
in-attempt failover to the next endpoint, probe-based failback.

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
where that is the convention). The transport options and behaviour (TLS/mTLS, TCP/UDP framing,
connection-liveness guards) are identical to the `syslog` output.

<a id="otlp-output"></a>
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
| `compression` | string | `gzip` | `gzip` or `none` |
| `encoding` | string | `protobuf` | **http only.** `protobuf` (binary OTLP) or `json` (proto3-JSON body). Setting `encoding` with `protocol: grpc` is a config error — gRPC is always protobuf on the wire |
| `batch_max_bytes` | int | `4194304` (4 MiB) | largest export request, measured as protobuf bytes before compression (with `protocol: http` and `encoding: json` the request body is larger than this) — the default receive limit of gRPC servers, the OpenTelemetry Collector included. A larger batch is sent as several requests; a single event over the limit can never be accepted and is rejected (`on_reject`) instead of being retried forever |
| `timeout` | duration | `10s` | per-request export deadline. When a request runs out of time, the next ones get double the deadline (up to 6 × `timeout`), and each success halves it back — a receiver that is slow but still accepting then gets to acknowledge instead of being sent the same batch again, which it would store twice |
| `on_reject` | string | `drop` | what happens to a batch the receiver **permanently** rejects (schema error, auth failure — a retry can never fix it): `drop` (log and discard) or `dead_letter` (quarantine to disk) |
| `dead_letter_dir` | string | `""` | required when `on_reject: dead_letter` — directory rejected batches are written to as proto3-JSON, one file per batch. **Capped at 100 MiB total**; once the cap is hit, further rejected batches are dropped (loudly logged) rather than growing the directory unbounded |

Forwards logs to any OTLP-compliant receiver — an OpenTelemetry Collector, or any backend with a native OTLP
ingest endpoint — over **gRPC** or **OTLP/HTTP**. Both protocols carry a structured `LogRecord` per event
(severity, timestamp + observed-timestamp, resource attributes for host/source, `syslog.procid`/`syslog.msgid`/
`syslog.facility` attributes) rather than a flattened text line — the natural fit for OTel-native platforms.
Batches are grouped into `resourceLogs` by `(host, source)`.

**Delivery semantics.** Unlike plain syslog/TCP, the output gets a **per-batch application-level acknowledgment**
from the receiver: gRPC status codes / HTTP status + the OTLP `partialSuccess` field tell the agent whether the
batch was actually accepted, not just TCP-received. An acknowledgment means the receiver *accepted* the batch: whether it had already stored it durably, or still held it in memory, is the receiver's design — a receiver that acknowledges from memory can still lose acknowledged events if it restarts or reloads before writing them, and the agent cannot see that. A **transient** failure (network error, HTTP `429`/`502`/
`503`/`504`, most gRPC codes) returns to the pipeline, which spools it to disk and retries — the same
store-and-forward guarantee as every other output. A **permanent** failure (gRPC `InvalidArgument`,
`NotFound`, `AlreadyExists`, `PermissionDenied`, `Unauthenticated`, `FailedPrecondition`, `Unimplemented` — the
codes that can only mean "the server made a final decision, retrying changes nothing") is never retried:
`on_reject` decides whether the batch is dropped or written to `dead_letter_dir` for offline inspection (the
100 MiB cap is a reliability bound, not automated replay — dead-letter is diagnostic quarantine). A partial
success (some records rejected, others accepted) is logged as a warning, per the OTLP spec, and not retried;
the rejected records count in `events_dropped`, the accepted rest in `events_out`. When a batch is sent as several
requests (`batch_max_bytes`) and one of them fails transiently, only the events not yet accepted are spooled and
retried: requests already accepted are not sent again, and an event rejected for its size is disposed of (and
written to `dead_letter_dir`) once.
Redirects are never followed. One bound applies to any acknowledged transport: an acknowledgment means the
receiver *accepted* the batch, so the agent's delivery guarantee ends at the receiver's front door — what
happens downstream of an ack is bounded by the receiver's own ack-to-durability depth. A receiver that
acknowledges into an in-memory queue can still lose acked batches if it crashes before flushing them onward;
where that matters, enable durable buffering on the receiver's onward path.

**Plaintext gRPC (h2c)** works out of the box against a stock `otel/opentelemetry-collector` OTLP receiver on
its default `:4317` — set `protocol: grpc` with `tls: false` for a LAN/dev collector, add `tls: true` (+
`ca_file`/mTLS) once TLS terminates in front of it. `encoding` only applies to `protocol: http`; gRPC is always
binary protobuf on the wire.

<a id="hec"></a>

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
| `source_style` | string | `agent` | `agent` sends the event's source as the agent names it (`file:/var/log/app.log`); `splunk` sends a file's bare path (`/var/log/app.log`), as a Splunk forwarder does, so existing `source=` searches keep matching. Other sources are unchanged |
| `event_format` | string | `message` | `message` (event text is the message body; structured fields ride as HEC *indexed fields*) or `json` (the whole structured event — message, severity, facility, host, source, fields — as the body) |
| `gzip` | bool | `false` | send POSTs with `Content-Encoding: gzip` (WAN-friendly) |
| `timeout` | duration | `10s` | per-request HTTP timeout |
| `batch_max_bytes` | int | `524288` (512 KiB) | batching granularity / memory bound. A single event larger than this can never be sent and is terminally rejected (dead-lettered + counted dropped) rather than retried forever |
| `use_ack` | bool | `false` | opt-in indexer acknowledgment — commit only once Splunk reports the batch **indexed** (the HEC token must also have acknowledgment enabled) |
| `ack_poll_every` | duration | `2s` | how often the agent polls Splunk for ack status in `use_ack` mode. Must not exceed `ack_timeout` (rejected at config-load) |
| `ack_timeout` | duration | `2m` | how long the agent waits for `ack=true` before treating the requests still unconfirmed as failed (only their events are spooled and retried) |
| `ca_file` | string | system roots | CA bundle to verify the server cert (`https://` endpoints only) |
| `insecure_skip_verify` | bool | `false` | **DEV ONLY** — disables server certificate verification (`https://` endpoints only) |
| `dead_letter_dir` | string | `""` | directory terminally-rejected (malformed) events are written to as newline-delimited JSON (NDJSON, one event object per line); omit to drop them (still counted). Capped at 100 MiB total — once full, further rejections are dropped and logged rather than growing the directory unbounded |
| `endpoints` | []string | `[]` | more HEC endpoints beside `endpoint` — other indexers or HEC tiers, same token and options, each in the same two URL shapes `endpoint` accepts. With more than one endpoint a batch that fails on one with a retryable error (connection failure, 5xx, server busy, authentication) is **retried at once on the next** instead of being spooled; only when every endpoint fails is it spooled and retried as before. A payload Splunk proves malformed is not re-sent elsewhere. In `use_ack` mode acknowledgements are polled on the endpoint that accepted the batch. Reported as `output_<name>_active_endpoint`, `output_<name>_failovers_total` and `output_<name>_failbacks_total`. Since 2.0.0 |
| `load_balance` | string | `round_robin` | with several endpoints: `round_robin` sends each batch to the next endpoint in turn, spreading load across indexers; `failover` always uses the first healthy endpoint in listed order (primary, then backups) and returns to the primary once it is healthy again |
| `retry_unhealthy_after` | duration | `30s` | how long an endpoint that just failed is tried only after the healthy ones. When every endpoint is failing, all are still tried for every batch |
| `max_in_flight` | int | `1` | how many batches may be on the way to Splunk at once (`1`–`64`). See below |

**Per-event metadata.** An event's [Splunk metadata fields](#splunk-metadata-fields) (`splunk.index`,
`splunk.sourcetype`, `splunk.source`, `splunk.host`) take precedence over `index` / `sourcetype` / `source`
above and over the event's host, and are never sent as indexed fields — so one output can deliver many
inputs, each to its own index and sourcetype. The HEC token must be allowed to write to every index named.

**Indexer acknowledgment.** HTTP-response mode and indexer-ack mode answer different questions: a 2xx
means Splunk's HEC endpoint *received* the batch, while `use_ack: true` waits for Splunk's ack poller to
confirm the batch was *indexed* — the stronger guarantee, but only available when both this flag is set
**and** the HEC token has acknowledgment enabled on the Splunk side (double opt-in; with `use_ack: false`
the output behaves identically against any standard token). While a batch is unacknowledged, the output
holds the pipeline batch — a batch larger than `batch_max_bytes` goes as several requests, each with its
own acknowledgment; requests confirmed within `ack_timeout` are committed, and only the events of the
requests still unconfirmed are spooled for retry. A request Splunk indexed but did not confirm in time can
then be **re-delivered and re-indexed as a duplicate** (at-least-once, not exactly-once — the same honesty
as every other output's reliability story). On a slow indexer, prefer **raising `ack_timeout`** over leaving it low: a too-tight timeout
converts ordinary indexing latency into repeated duplicate resends rather than actually protecting
anything. `ack_poll_every` must not exceed `ack_timeout` — the agent refuses to start with a poll interval
that can never fire before the deadline.

<a id="hec-max-in-flight"></a>
**Several batches on the way (`max_in_flight`).** By default the agent sends one batch, waits for
Splunk's answer (and, with `use_ack`, for the indexing acknowledgment), then sends the next — so a
receiver far away caps throughput at about one batch per round trip, however fast both ends are.
`max_in_flight: 4` keeps up to four batches on the way at once; raise it when the HEC endpoint is
remote or slow to answer (high latency, `use_ack` with a long indexing delay) and the agent cannot keep
up. Splunk may then receive batches in a different order than they were read; each event keeps its own
timestamp, and searches sort by it. Inputs still advance in read order, and delivery stays
at-least-once. Memory grows by up to that many batches. Leave it at `1` where nothing is slow; it
gains nothing then. Since 2.0.0

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
| `labels` | map | `{job: luna}` when neither `labels` nor `label_fields` is set | Static stream labels. **This is the safe place to put labels**. A stream that would otherwise carry no label at all also gets `job: luna` |
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
limit), any `5xx`, authentication failures, and ordinary network/timeout errors — is **retryable**: the events
stay in the disk spool and are retried, so a rate limit or a fixable credential never costs events. A batch
larger than `batch_max_bytes` goes as several pushes; when one of them fails, only its events are retried —
pushes Loki already accepted are not sent again, and a refused push is dead-lettered and counted once.

```yaml
- type: loki
  endpoint: "http://loki.example:3100"
  tenant: acme
  labels: { job: luna, env: prod }
  label_fields: [host]          # low cardinality only
  structured_metadata: true     # everything else rides here
  gzip: true
```

<a id="elasticsearch"></a>

### `beats` ✅ — Logstash / Graylog (Beats protocol) — **Apex**

Sends events with the Beats protocol (lumberjack v2 — what Filebeat and Winlogbeat speak) to **Logstash's `beats`
input**, **Graylog's Beats input**, or another agent's `beats_in` input.
Use it where Logstash is already the collection tier, so the agent replaces a Beat without changing the pipeline
behind it.

| Option | Type | Default | Notes |
|---|---|---|---|
| `endpoint` | string | *(required unless `endpoints`)* | `host:port`, e.g. `logstash.example:5044` |
| `endpoints` | []string | `[]` | further receivers, used in order when the current one fails (failover; the agent stays on the one that works) |
| `timeout` | duration | `30s` | how long to wait for the receiver's next acknowledgement; keepalive acknowledgements and partial progress restart it, so a slow receiver that is still working keeps the batch open |
| `compression_level` | int | `3` | zlib level 1–9, `0` = uncompressed (Beats' default is 3) |
| `index` | string | `luna` | sent as `@metadata.beat` — Logstash's default Elasticsearch index pattern is `%{[@metadata][beat]}-…` |
| `tls` | bool | `false` | connect with TLS (also implied by `ca_file` / `cert_file` / `cert_source: enrolled`) |
| `ca_file` / `cert_file` / `key_file` / `server_name` / `insecure_skip_verify` / `cert_source` | | | TLS / mTLS, as on the [`syslog` output](#syslog) |

**Delivery:** each batch is one window (a batch that would encode to more than 64 MiB is sent as several), and a
write counts as delivered only when the receiver has acknowledged every event in it; anything less (a dropped connection, a timeout) leaves the batch for the spool and a retry.
When a batch is sent as several windows and one fails, only the windows not acknowledged are retried; an event
too large for any window is dropped and counted once. A window that was partly acknowledged before a failure is
resent whole, so expect occasional duplicates after a receiver restart (at-least-once).

**Document:** ECS-shaped JSON — `@timestamp`, `message`, `host.name`, `log.level` (the syslog severity name),
`log.syslog.severity.code`, `log.syslog.facility.code`, `event.module` (the agent input that read the event,
e.g. `file:/var/log/app.log`), `agent.type: luna`, `agent.version`, and every event field under its own dotted
path as nested objects (`winlog.event_id` arrives as `[winlog][event_id]`). When one field name is also the prefix
of another (`a` and `a.b`), the second keeps its full dotted name as a key so neither value is lost.

### `fluent` ✅ — Fluentd / Fluent Bit (Forward protocol) — **Apex**

Sends events with the Fluent Forward protocol to a **Fluentd** or **Fluent Bit** `forward` input — typically an
aggregator tier — or to another agent's `fluent_in` input.

| Option | Type | Default | Notes |
|---|---|---|---|
| `endpoint` | string | *(required unless `endpoints`)* | `host:port`, e.g. `fluentd.example:24224` |
| `endpoints` | []string | `[]` | further receivers, used in order when the current one fails (failover) |
| `tag` | string | `luna` | tag for events that do not carry one; an event received by `fluent_in` keeps its original tag (`fluent.tag`) |
| `shared_key` | string | `""` | answer the receiver's shared-key handshake (Fluentd `<security>`, Fluent Bit `Shared_Key`); the receiver must prove it knows the key too |
| `require_ack` | bool | `true` | wait for the receiver's acknowledgement of each request (Fluentd/Fluent Bit acknowledge requests that ask for it); `false` = fire-and-forget |
| `timeout` | duration | `60s` | how long to wait for an acknowledgement |
| `compression` | string | `gzip` | `gzip` (CompressedPackedForward) or `none` |
| `tls` | bool | `false` | connect with TLS (also implied by `ca_file` / `cert_file` / `cert_source: enrolled`) |
| `ca_file` / `cert_file` / `key_file` / `server_name` / `insecure_skip_verify` / `cert_source` | | | TLS / mTLS, as on the [`syslog` output](#syslog) |

**Delivery:** one request per tag per batch; a write counts as delivered only when every request was acknowledged.
A batch whose entries exceed 8 MiB is sent as several requests; a single event over that can never be accepted
and is dropped and counted (once). Failures leave the events not yet acknowledged for the spool and a retry
(at-least-once); requests already acknowledged are not sent again.

**Record:** every field under its dotted path as nested maps (`kubernetes.pod_name` arrives as `kubernetes` →
`pod_name`), plus `message`, `host`, `level` (syslog severity name) and `source` (the agent input) — each of those
four under `luna` instead when the event already has a field of that name, so nothing is replaced. The event time is
sent with nanosecond precision. With several `endpoints`, the output returns to the first one 60 s after failing
over.

### `elasticsearch` ✅ — Elasticsearch and OpenSearch (bulk API) — **Apex**

Indexes events into **Elasticsearch** or **OpenSearch** through the `_bulk` API — the same API on both, so one
output serves either. Every event is a `create` action, which data streams require; the default index
`logs-luna-default` matches Elasticsearch's built-in `logs-*-*` data-stream template, and on a cluster
without that template it becomes an ordinary index.

Each event becomes one document in Elastic Common Schema shape: `@timestamp` (the event's own time),
`message`, `host.name`, `log.syslog.severity.code`, `log.syslog.facility.code`, `event.provider` (the source,
e.g. `file:/var/log/app.log`), `event.created` (when the agent received it), `agent.type` / `agent.version`, and
the event's fields under `labels` — with dots in field names replaced by `_`, because Elasticsearch would
otherwise expand a key such as `snmp.oid.1.3.6…` into nested objects.

Delivery: the response is checked per document. Documents the cluster throttles or fails on (429, 5xx) are
**retried within the same delivery, and only those documents** — `item_retries` times with backoff — so a
partial throttle does not duplicate the documents that already landed; if some still fail, only those
documents are spooled and resent, and the same holds when a whole request fails part-way through a batch
sent as several requests — documents already indexed are not sent again. Documents the
cluster refuses for good (400, e.g. a mapping conflict; 404, e.g. a missing index with auto-create off) are
dead-lettered or dropped, counted as `output_rejected`, and the rest of the batch is delivered. 409 (the
document already exists) counts as delivered. A whole request that fails (connection error, 5xx, 401/403,
413, 429) is tried on the next of `endpoints`, then spooled.

| Option | Type | Default | Notes |
|---|---|---|---|
| `endpoints` | []string | *(required)* | one or more `http(s)://host:9200` node URLs; requests rotate across them and a failing node is skipped |
| `index` | string | `logs-luna-default` | data stream or index name (lowercase; no spaces or `\ / * ? " < > \| , # :`; not starting with `_` or `-`) |
| `api_key` | string | `""` | API key as `id:key` or its base64 form; sent as `Authorization: ApiKey …`. Mutually exclusive with `username` |
| `username` | string | `""` | basic-auth user (with `password`) |
| `password` | string | `""` | basic-auth password |
| `timeout` | duration | `30s` | per-request HTTP timeout |
| `batch_max_bytes` | int | `5242880` (5 MiB) | largest bulk request body; a larger batch is sent as several requests (clusters cap requests at 100 MB by default) |
| `item_retries` | int | `3` | how many times documents that failed with a retryable status are resent within one delivery before those documents are spooled |
| `ca_file` | string | system roots | CA bundle to verify the cluster's certificate (`https://` endpoints) |
| `insecure_skip_verify` | bool | `false` | **DEV ONLY** — disables certificate verification |
| `dead_letter_dir` | string | `""` | where documents refused for good are written as newline-delimited JSON (100 MiB cap); empty = drop them, counted |

```yaml
- type: elasticsearch
  endpoints: ["https://es1.example:9200", "https://es2.example:9200"]
  index: logs-edge-prod
  api_key: "REPLACE_WITH_ES_API_KEY"
```

### `kafka` ✅ — Apache Kafka / Kafka-compatible brokers (Redpanda, …) — **Apex**

Produces events to a Kafka topic — the sink to reach for when the destination is a Kafka-based
pipeline or data lake rather than a syslog receiver. Acked like `otlp`/`hec`: the producer's
per-record delivery report commits each record, transient broker failures (down broker, leader
election, throttling) spool-and-retry with at-least-once delivery — only the records that failed are
produced again, never those the broker already acknowledged — and idempotent produce is
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
    ca_file: /etc/logrok-universal-agent/kafka-ca.pem
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
  #   staging_dir: /var/lib/logrok-universal-agent/s3-staging
  #   # flush_interval: 5m
```

### `sentinel` ✅ — Microsoft Sentinel / Azure Monitor Logs (Logs Ingestion API) — **Apex**

Sends events to a Log Analytics workspace — and thereby Microsoft Sentinel — via the modern **Logs
Ingestion API** with a Data Collection Rule (DCR). Batches are posted as JSON arrays (gzipped by
default, split automatically under the API's per-call size bound); the service's success response
commits that call's rows, and when one call of a batch fails only its rows are retried. Transient failures (network, 5xx, throttling) spool-and-retry with at-least-once
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
commits that request's records, and when one request of a batch fails only its records are retried.
Transient failures, throttling, wrong-endpoint and authentication failures all
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

### `relay` ✅ — ack'd reliable transport (agent→agent) — **Apex**

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
| `compress` | bool | `false` | gzip-compress the batches sent to the receiver |
| `ack_timeout` | duration | `10s` | per-attempt wait for the receiver's acknowledgment |

The agent's answer to syslog-ng PE's proprietary **ALTP**: an acknowledged protocol over TCP/TLS. A batch
counts as delivered **only after the receiver acknowledges the batch** — the application-level ack a plain TCP write can't give —
so the disk spool + retry now covers the *peer*, not just the wire: a TLS-rejected client cert, a receiver
crash mid-batch, or a connection drop before the ACK all land back in the spool. On a dropped connection the
output reconnects and resends the batch, which the receiver recognises and does not store twice — a lost
acknowledgment never duplicates events. A receiver **NACK** fails the write immediately (re-spool, no blind retry). Does **not**
replace plain syslog: the receiver must be a `relay_in` agent; the final hop to any third-party aggregator
stays standard RFC 5424.

---

## Minimal examples

**Linux gateway — receive syslog, forward over mTLS:**
```yaml
service: { mode: standalone, log_level: info }
buffer:  { enabled: true, dir: /var/lib/logrok-universal-agent/spool, max_bytes: 1073741824 }
inputs:
  - { type: syslog_in, listen: "0.0.0.0:5514", protocols: [tcp, udp] }
outputs:
  - type: syslog
    endpoint: "aggregator:6514"
    tls: true
    ca_file: /etc/logrok-universal-agent/ca.pem
    cert_file: /etc/logrok-universal-agent/agent.pem
    key_file: /etc/logrok-universal-agent/agent.key
```

**Diode mode — one-way link (data diode / unidirectional gateway) out of an OT zone** (UDP diode mode is an **Apex** capability):
```yaml
# Send-only profile: UDP (diodes pass UDP only), sequence SD for receiver-side
# gap detection, disk spool so a NIC/link outage loses nothing locally, and no
# management endpoint (nothing can reach back through a one-way link).
service: { mode: standalone, log_level: info }
buffer:  { enabled: true, dir: /var/lib/logrok-universal-agent/spool, max_bytes: 1073741824, when_full: block }
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
