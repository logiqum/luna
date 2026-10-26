# CMMC 2.0 Level 2 — NIST SP 800-171 Audit & Accountability (AU / 3.3.x) mapping

CMMC 2.0 Level 2 (most DoD contractors handling CUI) assesses the NIST SP 800-171 controls; the AU family
(3.3.1–3.3.9) is the logging core. Enforcement phases in through 2025–2026 contract cycles.
Sources: [ISI Defense — CMMC logging](https://isidefense.com/blog/cmmc-continuous-monitoring) ·
[Kiteworks — CMMC 2.0 AU checklist](https://www.kiteworks.com/best-practices-checklist-cmmc-2-0-audit-and-accountability-requirement/) ·
NIST SP 800-171r2 control text. *Not compliance advice — confirm scoping with your C3PAO.*

| Control | What it asks | Agent today | Notes / whose job |
|---|---|---|---|
| **3.3.1** create & retain audit logs | System audit logs/records to enable monitoring, analysis, investigation | ✅ collects the OS-level trail (Windows Event Log incl. Security (Apex); journald; the Linux kernel audit trail via `linux_audit` (Apex) — process executions, file-access watches, logins; files) and delivers it to the retention store without gaps (store-and-forward, with the disk spool enabled) | Retention duration = platform |
| **3.3.2** trace to individual users | Actions traceable to unique users | ✅ passes through the OS identity fields (Windows logon events, SIDs/usernames in EventData; journald `_UID`/unit fields; `linux_audit` (Apex) UID/AUID fields) as structured SD | Identity hygiene (unique accounts) = yours |
| **3.3.3** review/update logged events | Periodically re-decide *what* you log | ◐ per-module config + XPath `query`/processors make the logged-event set explicit, reviewable, and centrally changeable (mgmt plane: config-pull — already shipped) | The review itself = yours |
| **3.3.4** alert on audit-process failure | Alert when audit logging fails | ✅ the strongest fit: `/metrics` (`events_dropped_total`, buffer depth, `last_forward_timestamp_seconds`) + heartbeat make collection failure observable and alertable | Alert rules = monitoring stack |
| **3.3.5** correlate review/analysis | Correlation across repositories | ◐ feedstock: structured fields, single wire format | Correlation = platform |
| **3.3.6** reduction & report generation | On-demand analysis/reporting | ⛔ platform-side (logrok SQL) | — |
| **3.3.7** authoritative time | Timestamps from synchronized authoritative source | ◐ preserves source timestamps as RFC 3339 **UTC**; does not sync clocks (OS/NTP) | — |
| **3.3.8** protect audit info | Protect logs from unauthorized access/modification/deletion | ● keyed HMAC-SHA-256 hash-chained spool (the chain is on by default once the disk spool is enabled; the spool itself is opt-in: it needs both `buffer.enabled` and `buffer.dir`, and is an **Apex** capability) + offline `-verify-spool` auditor check; detection not prevention — see limitations | Also off-host within seconds (shrinks local tamper window); TLS/mTLS in transit. Limitations (see [compliance README](README.md)): a key-reader attacker can re-forge the chain; un-anchored tail window; `head.json`/whole-spool deletion degrades detection. At-rest protection = platform disk encryption, plus opt-in AES-256-GCM encryption of the agent's own spool (`buffer.encrypt`) |
| **3.3.9** privileged log management | Limit audit management to privileged subset | ◐ agent runs as a service with a fixed config; config changes go through OS-privileged file access (or the authenticated mgmt plane, already shipped) | — |

**Defense-context notes:** static no-call-home binary suits disconnected/classified-adjacent enclaves;
**FIPS 140-3 mode is the shipped default** on every binary except OpenBSD and AIX (where the module does not
support it): cryptography comes from the Go Cryptographic Module
v1.0.0 (CMVP certificate #5247), reported under `-version` and on `/metrics`. `service.fips_mode: required` refuses
to start unless the agent binary is running in FIPS 140-3 mode (it checks the agent process, not the host operating system); `service.fips: off` is the escape for a receiver that offers
neither TLS 1.3 nor Extended Master Secret. The validated component is the module — whether that satisfies your
assessor is their call, not a claim we make.
