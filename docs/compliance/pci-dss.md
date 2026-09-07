# PCI DSS v4.0 — Requirement 10 mapping

Requirement 10: "Log and monitor all access to system components and cardholder data." The
formerly-best-practice items became **mandatory 2025-03-31** (v4.0 future-dated requirements took effect
March 31, 2025). Source: [PCI DSS Requirement 10 guide](https://pcidssguide.com/pci-dss-requirement-10/).
*Not compliance advice — confirm with your QSA.*

| Req | What it asks | Agent today | Notes / whose job |
|---|---|---|---|
| **10.2** Audit events | Log access to cardholder data, admin actions, auth success/failure, log start/stop, object create/delete | ✅ `windows_eventlog` (Security channel: logon/logoff, account & policy changes) + **Sysmon preset** (process creation w/ command line + hashes, file/registry events) + `linux_audit` (kernel audit trail: process executions, file-access watches, logins) + `journald`/`filetail` for Linux/app trails | Which systems are in CDE scope is yours; the agent collects whatever endpoints you deploy it on |
| **10.3** Protect logs | Limit access; tamper-evidence / change detection on logs | ● keyed HMAC-SHA-256 hash-chained spool (on by default) + offline `-verify-spool` auditor check; detection not prevention — see limitations | Prompt off-host forwarding also **shrinks the local tampering window** (events leave the endpoint seconds after they occur). Limitations (see [compliance README](README.md)): an attacker who can read the chain key can re-forge it; records written after the last persisted anchor have an un-anchored tail window (modification-detectable, not clean-truncation-detectable); `head.json`/whole-spool deletion degrades detection. Access control over *stored* logs = platform/SIEM |
| **10.4** Review | Automated daily log review (mandatory since 2025-03-31) | ◐ Agent's job is the feedstock: complete, structured (RFC 5424 SD fields), UTC-stamped events | Review/alerting itself = logrok / your SIEM |
| **10.5** Retention | 12 months total; 3 months immediately accessible | ◐ Agent contributes **no-gap delivery**: store-and-forward rides out WAN/VPN outages (typical retail-branch failure mode) and drains automatically | Retention itself = platform storage |
| **10.6** Time sync | Synchronized, correct time (NTP) on all CDE systems | ◐ Agent **preserves source timestamps** in UTC (RFC 3339) end-to-end | Clock sync = OS/NTP duty; skew on the endpoint propagates into any product's logs |
| **10.7** Detect control failures | Detect, alert, and respond to failures of critical security controls promptly | ✅ `/metrics` exposes `events_in/out_total`, **`events_dropped_total`**, buffer depth, `last_forward_timestamp_seconds` + heartbeat — a silent collection failure is visible within one scrape/heartbeat interval | Alert rules on those metrics = your monitoring (logrok ships Prometheus/Grafana) |

## Requirement 11.5 — change-detection mechanisms (file integrity)

Requirement 11.5 asks for a change-detection mechanism over critical files, with comparisons at least
weekly and alerts on unauthorised modification. **The agent is the feedstock for one, not the mechanism.**
Read this row as a boundary, not a claim.

| Req | What it asks | Agent today | Notes / whose job |
|---|---|---|---|
| **11.5.2** Change-detection mechanism | Deploy a mechanism that alerts on unauthorised modification of critical files; compare at least weekly | ◐ Forwards file-change **events** with the actor attached — `linux_audit` (auditd watch rules: writes and attribute changes on watched paths, with uid/auid/pid/command line) and `windows_eventlog` (Sysmon FileCreate/FileDelete/RegistryEvent, or native object-access auditing 4663/4656/4670 with SACLs). Delivered with store-and-forward and a tamper-evident spool, so an outage does not become an evidence gap | **The comparison and the alerting are not the agent's.** It can attach a changed file's content digest via the `file_hash` processor (an **Apex**-tier capability — see [LICENSING](../LICENSING.md)), but it holds no baseline, runs no scheduled scan, and makes no authorised-vs-unauthorised judgement — that is your platform/SIEM (logrok, where the rule corpus lives). The agent also does not configure the OS audit subsystem: **you** load the auditd rules or set the SACLs, and with none loaded there is simply no traffic. **macOS is not covered.** See [USER-GUIDE § File-integrity events](../USER-GUIDE.md#file-integrity-events-fim--what-the-agent-does-and-what-it-deliberately-doesnt) |

**Deployment pattern (retail branch):** Windows endpoints + POS with `buffer.enabled: true`,
`when_full: block` if zero-loss matters more than freshness, TLS/mTLS to the aggregator. See
[USER-GUIDE § Air-gapped / intermittent links](../USER-GUIDE.md#air-gapped--intermittent-links).
