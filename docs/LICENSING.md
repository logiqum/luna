# LUnA — Licensing & entitlements

How the agent decides which capabilities it may run. The model is **offline-first**
(no phone-home), so it works in air-gapped and OT environments.

> **The binding terms are in [`LICENSE`](../LICENSE), not in this document.** This guide explains the
> *mechanics* — how entitlements are delivered, verified, and enforced. What you are permitted to run,
> and on how many agents, is stated in the End-User License Agreement itself. Where the two ever appear
> to differ, `LICENSE` governs; please read it rather than relying on any summary, including this one.

## 1. The model

Two orthogonal axes:

- **Context** — the free-vs-paid axis: `non_commercial` (free), `bundled_with_logrok`
  (free), `commercial_standalone` (paid).
- **Tier** — the capability tier, two only: **Core** (free) → **Apex** (paid).

The free **Core** tier always runs with **no license at all**; every gated capability is **Apex**:

| Tier | Contents |
|---|---|
| **Core** (free) | file tail · syslog-in · journald · HTTP-in · parsers (json/csv/kv/xml/expr/add_fields/filter) · TLS syslog output · metrics |
| **Apex** (paid) | **Windows Event Log** · ETW · WMI · Linux audit (`linux_audit`) · MQTT ingest (`mqtt_in`) · disk store-and-forward spool · central fleet management · edge reduction (sample/throttle/dedup/trim_fields/quota/adaptive_sample) · redaction (`redact`) · enrichment (`lookup`) · per-agent enrolled mTLS · ack'd relay transport · Kubernetes container-log collector · macOS unified-log (`oslog`) · UDP / data-diode output · **multi-destination fan-out** (more than one output) · SNMP trap/inform receiver (`snmptrap_in`) · Modbus (`modbus_in`) · **every destination other than RFC 5424 syslog** (`snare`/`hec`/`sentinel`/`xsiam`/`kafka`/`s3`/`loki`/`otlp`) · extended OS platforms (AIX/Solaris/BSD) |

**The dividing line, in one sentence:** Core collects from the host and forwards **one open
standard — RFC 5424 syslog over TCP/TLS — to one destination**. Anything that reads a privileged or
proprietary OS subsystem, or speaks a named vendor or platform API, is Apex. TLS is never gated:
encryption is not a paid feature.

### Modules newly marked Apex (notice period)

Fifteen modules were Apex by this definition but were not marked or checked in earlier releases:
`etw`, `wmi`, `linux_audit`, `mqtt_in`, `redact`, `lookup`, `adaptive_sample`, and the eight
non-syslog outputs (`snare`, `hec`, `sentinel`, `xsiam`, `kafka`, `s3`, `loki`, `otlp`).

They are now marked and reported, but **not withdrawn**: an unlicensed agent using them warns, reports
posture `degraded`, and keeps running them until the next **major** version. See §6. **Core did not
change** — nothing that was free has been removed from it.

Legacy tier names from earlier license files (`orbit`, `supra`) still parse and resolve to Apex.

## 2. The two entitlement sources

1. **Bundled with logrok (free) — enrollment-derived.** A logrok-managed agent receives
   a signed entitlement from the control plane at enrollment; the agent persists it in its
   state file and uses it automatically.
2. **Standalone commercial — a signed license file.** Point the agent at a `.lic`:
   ```yaml
   licensing:
     license_file: /etc/logrok-agent/agent.lic
   ```
3. **No license → free Core.** Omit both and the agent runs Core (also the non-commercial path).

**Precedence:** enrollment-derived (the control plane is authoritative for managed agents) →
`license_file` → Core.

## 3. What a license file *is*

A small JSON envelope: a **payload** (the entitlement) plus an **Ed25519 signature** over
those exact payload bytes, verifiable offline against the issuer public key embedded in the
agent binary. It is **not** encrypted — it's signed (tamper-evident), so it's fine to read.

```json
{
  "payload": { "context": "commercial_standalone", "tier": "apex",
               "customer": "ACME GmbH", "license_id": "5f3c…",
               "issued_at": "2026-06-28T00:00:00Z", "expiry": "2027-06-28T00:00:00Z",
               "graced_until": "2027-07-28T00:00:00Z",
               "capabilities": [], "agent_limit": 0 },
  "signature": "<base64 ed25519 over the payload bytes>",
  "algorithm": "ed25519",
  "key_id": "logrok-v1"
}
```

Entitlement fields: `context`, `tier` (`core` or `apex`), optional `capabilities[]`
(grant individual features above the tier, e.g. a Core+macOS add-on), `agent_limit`
(0 = unlimited), `expiry` (zero/omitted = perpetual), `graced_until` (optional issuer-set
end of the grace window after `expiry` — see §6; omitted = a default 30-day grace), and
informational `customer`/`license_id`/`issued_at`.

## 4. Where it lives

| Source | Location |
|---|---|
| Standalone license file | wherever `licensing.license_file` points (convention: `/etc/logrok-agent/agent.lic`, or `C:\ProgramData\logrok-universal-agent\agent.lic`). Operator-managed; survives upgrades. |
| Enrollment-derived | persisted inside the agent **state file** (`agent-state.json`, next to the config by default) as the `entitlement` field — written at enrollment, re-verified offline on every load. |
| Issuer **public** key | embedded in the agent binary. |
| Issuer **private** key | **never in the agent** — held by the vendor. Only the holder of the private key can mint a license. |

## 5. Getting a license

The free **Core** tier and the non-commercial / bundled-with-logrok paths need **no license
file**. A **standalone commercial** license is **issued by the vendor** and verified by the
agent **offline** (no phone-home), so it works in air-gapped deployments. Drop the issued
`.lic` at the `licensing.license_file` path above and (re)start the agent.

## 6. Enforcement: license states, grace, and degrade-to-Core

The agent evaluates its license posture at **config-apply** (every start / config reload):
it maps the config to the gated capabilities it uses and compares them to the active
entitlement. Two invariants hold in every state:

- **Licensing never crashes or stops the agent.** The worst outcome is running with fewer
  features, never not running.
- **Core capabilities are never blocked.** An absent, invalid, or expired license means
  Core — the same as no license.

The four states (visible on `/metrics` as `logrok_agent_license_state` and, for managed
agents, on every heartbeat):

| State | Meaning | Behavior |
|---|---|---|
| `core` | the config uses no Apex capability | nothing to license; runs as-is |
| `licensed` | a valid entitlement covers every Apex capability in use (paid or active trial) | runs as-is |
| `grace` | the entitlement covered these capabilities but has expired, and the grace window is still open | Apex features **keep running**, with an escalating warning and a day counter (`logrok_agent_license_grace_days_left`). The grace window ends at the license's `graced_until`, or 30 days after `expiry` if unset |
| `degraded` | the grace window has ended, or the entitlement never covered these capabilities | the unlicensed Apex modules are **withdrawn** (degrade-to-Core) — the disk spool as *drain-only*, see below; everything Core keeps running. Installing a valid license and restarting (or pushing a config reload) restores them immediately. **Exception — the notice period (§1):** a module inside its notice period is *reported* in this state but is **not** withdrawn; it keeps running until the next major version. A config using only those modules therefore reads `degraded` while its pipeline is entirely intact — that is the warning, and it is deliberate: reporting `licensed` would say the entitlement covers something it does not |

What degrade-to-Core does, concretely: the unlicensed Apex inputs/processors/outputs are
removed from the running pipeline, each removal logged at startup
(`licensing: degraded to Core — unlicensed Apex features disabled`). An unlicensed
multi-output (fan-out) config keeps its **first** output and drops the rest — forwarding
never stops entirely. A few capabilities are
warned about but never disabled, because disabling them would be self-defeating: the
fleet-management/enrollment plumbing (an agent must keep the path to *becoming* licensed) and
the host platform itself (extended OS builds).

**The disk spool is withdrawn as *drain-only*, never simply switched off.** Anything already
written to the spool is still delivered — the agent does not abandon events it has already
accepted — and only *new* events are refused. You will see one warning at startup and a single
`spool drain complete` line once the backlog has been delivered. Ordinary forwarding is
unaffected while it drains; events are lost only if an output is unavailable with no buffering
left to catch them, and each such loss is counted in `logrok_agent_events_dropped_total` and
logged rather than passing silently. The spool directory and its contents are never deleted, so
installing a licence restores buffering with the data intact.

A config that is *entirely* Apex (nothing Core to fall back to) degrades to an idle-but-alive
agent — running, manageable, forwarding nothing — rather than exiting.

**Enforcement exceptions:** for controlled situations (e.g. a migration window agreed with
the vendor), a temporary enforcement exception can be arranged — contact support. Unlicensed
Apex use is always detected and logged, whatever the enforcement posture.
