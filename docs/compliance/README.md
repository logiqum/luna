# Compliance mapping — how the agent supports audit & regulatory log requirements

One page per framework, mapping **requirement → what the agent provides today → what's on the roadmap →
what stays the operator's / SIEM's job**. These are purchase-justification and audit-prep artifacts: they
deliberately do *not* overclaim. Where a control is met by the platform (logrok or any SIEM) or by the
operator, the doc says so.

> **Not legal or compliance advice.** Requirements are summarized from the cited primary/secondary sources
> as of 2026-06. Verify against the current standard text and your assessor's interpretation.

> **Licensing note.** These mappings describe the agent's capability, not its free tier. Several of the
> capabilities cited — Windows Event Log, the Linux kernel audit trail (`linux_audit`), redaction, disk
> store-and-forward, fleet management — are **Apex** (paid, and included at no charge with a licensed
> logrok deployment). Budget for a licence when scoping a control to them; see
> [LICENSING.md](../LICENSING.md) for the full Core/Apex split.

| Framework | File | Who it's for |
|---|---|---|
| PCI DSS v4.0 Requirement 10 | [pci-dss.md](pci-dss.md) | retail / payments / anyone in CDE scope |
| NERC CIP-007-6 R4 | [nerc-cip.md](nerc-cip.md) | North American bulk-electric utilities |
| NIS2 (EU 2022/2555) | [nis2.md](nis2.md) | EU essential/important entities (18 sectors) |
| DORA (EU 2022/2554) | [dora.md](dora.md) | EU financial entities + ICT providers |
| CMMC 2.0 L2 / NIST SP 800-171 AU | [cmmc.md](cmmc.md) | US DoD contractors handling CUI |

HIPAA (45 CFR §164.312(b) audit controls; 6-year documentation retention) follows the same shape as PCI —
collect OS-level audit trails, deliver without gaps, retain downstream — and is covered by the same agent
capabilities; a dedicated page can follow if healthcare deals need it.
([Kiteworks HIPAA audit-log guide](https://www.kiteworks.com/hipaa-compliance/hipaa-audit-log-requirements/))

## The agent capabilities these mappings draw on

- **Collection:** Windows Event Log incl. the Security channel (admin/`SeSecurityPrivilege` handled
  gracefully) and **Sysmon** (`configs/sysmon.example.yaml`); file tail; journald; the Linux kernel audit
  trail (`linux_audit` — process executions, file-access watches, logins); syslog-in.
- **No-gap delivery:** disk store-and-forward (CRC-checked, bounded, `when_full` policies incl. `block`
  = zero-drop backpressure), at-least-once forwarding, automatic drain after outages.
- **Tamper-evident spool (shipped 2026-07-05):** the disk spool is a keyed HMAC-SHA-256 hash chain over
  every record and segment, **on by default** with an auto-generated local key — no operator setup
  required. Detection (not prevention): modification, deletion (including front-truncation — deleting the
  oldest segments of a backlog), insertion, or reordering of already-written spool content is evidenced,
  both live during drain (`spool_tamper="true"` on the affected event, a metric, a sticky heartbeat flag)
  and offline via the `logrok-universal-agent -verify-spool` auditor CLI (verdicts `INTACT` /
  `TAMPER EVIDENCE` / `UNVERIFIABLE`, scriptable exit codes). See
  [USER-GUIDE.md § Auditing the spool](../USER-GUIDE.md#auditing-the-spool) for how to run it.
- **Signed export manifests (shipped 2026-08-16) — verifiable chain-of-custody off the host.** The chain
  above verifies only where its secret key lives; the export manifest extends the evidence across a
  transfer: `-export-spool` attests the exact spool file set (SHA-256 per file) plus the chain verdict at
  export time, Ed25519-signed with a per-agent key, and `-verify-export` checks a couriered/copied bundle
  anywhere with only the agent's **public** key and the same binary (verdicts `VERIFIED` /
  `EXPORT MISMATCH` / `BAD SIGNATURE` / `UNVERIFIABLE`, scriptable exit codes). Covers the audit-evidence
  hand-off: air-gapped courier transport, forensic preservation, and evidence bundles. See
  [USER-GUIDE.md § Exporting a verifiable bundle](../USER-GUIDE.md#exporting-a-verifiable-bundle-signed-manifest).
- **Transport security:** TLS with verification on by default; mTLS client certs.
- **Timestamps:** RFC 5424 / RFC 3339 **UTC** preserved from the source event.
- **Self-evidence:** Prometheus `/metrics` (events in/out, **dropped**, buffer depth, last-forward time,
  spool tamper detections) + heartbeat — the "is collection actually working" signal auditors ask about.
- **Constrained links:** UDP "diode mode" with `sequence` SD for receiver-side gap detection; fully
  unmanaged (air-gapped) operation from a local config file.

**Honesty notes (claims we deliberately do NOT make):**
- The chained spool is **detection, not prevention**: an attacker with agent-level privileges who can read
  the local chain-key file can re-forge a self-consistent chain. The mitigation is the product's existing
  posture (prompt off-host forwarding) plus keeping the key **outside** the spool directory by default, so
  an offline copy of the spool doesn't carry its own key.
- **The export manifest's trust root is still the agent host:** an attacker who owns the host owns the
  Ed25519 signing key and can forge manifests. The manifest makes evidence verifiable *off*-host; it does
  not shrink the on-host trusted base. It also attests the exported **set**, not completeness — events
  that never reached the spool are outside any spool-level evidence.
- **Un-anchored tail window:** records appended since the last persisted anchor (segment rotation, ack, or
  clean close) are modification-detectable but not clean-truncation-detectable — an attacker can snip the
  chain's very end without evidence. Everything behind the last persisted anchor is fully covered.
- **`head.json` (or whole-spool) deletion degrades, doesn't fool, the verifier:** losing `head.json` reports
  as "anchors unavailable" (truncation detection lost) rather than a false-clean verdict — an absent anchor
  is not treated as evidence of tampering, only garbled anchor *values* are. Deleting the entire spool
  directory is self-evident (it's simply gone) and outside what any chain can detect.
- **Front-truncation of already-acknowledged segments** (normal FIFO consumption deleting fully-sent
  segments) is indistinguishable from legitimate drain behavior — inherent, not a gap. (Front-truncation of
  the backlog — an attacker deleting oldest segments that were *not* yet acked — is detected, per above.)
- **Anchor rollback to a consistent earlier state — now detected via the recency witness.** `head.json`
  carries a monotonic **generation** (bumped each time the agent opens the spool, authenticated by the
  same key as the chain) and the agent keeps an out-of-spool copy beside its chain key, plus reports it
  on every heartbeat when centrally managed. An attacker who restores an earlier authentic snapshot of
  the spool directory presents a chain that verifies — but its generation is older than the witness, and
  the agent raises rollback evidence at the next start (and the platform can flag the decrease across
  heartbeats). The auditor accepts the reference too: `-verify-spool <dir> -recency <spool-recency.json>`
  reports `ROLLBACK EVIDENCE`; without a reference it reports recency as **unverified** rather than
  implying it. Remaining residual, stated plainly: an attacker who controls BOTH the spool directory and
  the agent's state directory — and whose agent is unmanaged (no heartbeat) — can restore both halves
  coherently; only off-host forwarding and the platform-side check cover that case. Modification,
  insertion, reordering, and truncation that does NOT
  match a held earlier anchor remain detected.
- **Legacy-disguise:** an attacker who strips the chain framing back to the old unchained format on-disk
  evades the *runtime* flags (which gracefully drain legacy segments so upgrades don't wedge) — but the
  *offline* verifier still reports those segments as unverifiable-legacy. Unexpected legacy segments on a
  spool that should be fully chained are an audit finding.
- **Runtime does not verify cross-segment ancestry** (a deleted middle segment is silent during live drain);
  the offline `-verify-spool` is the authoritative check — run it, don't rely on the running agent alone.
- **The heartbeat `spool_tamper` flag is sticky, not a live status:** it latches on first detection and
  stays `true` until restart or a config hot-reload rebuilds the management client. The durable evidence is
  the in-band `spool_tamper="true"` event field and the offline verifier's report, not the heartbeat flag.
- The agent does not synchronize clocks (OS/NTP duty) and does not itself retain logs for N months
  (platform duty) — it preserves source timestamps and delivers without gaps.
- Retention, automated review, alerting, and access control over stored logs are **platform/SIEM
  controls** (logrok or any aggregator you forward to).
