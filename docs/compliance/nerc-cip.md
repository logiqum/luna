# NERC CIP-007-6 — R4 Security Event Monitoring mapping

CIP-007-6 Table **R4** requires documented processes for security event monitoring on BES Cyber Systems
(+ associated EACMS/PACS/PCAs as applicable). R4-related violations — particularly log retention — are
actively enforced ([2024 NERC enforcement actions](https://www.nerc.com/programs/enforcement/enforcement-actions/2024)).
Primary text: [CIP-007-6 standard (PDF)](https://www.nerc.com/globalassets/standards/reliability-standards/cip/cip-007-6.pdf).
*Not compliance advice — confirm with your compliance team/auditor.*

| Part | What it asks | Agent today | Notes / whose job |
|---|---|---|---|
| **4.1** Log events | Log security events per BES Cyber System capability — at minimum detected successful logins, failed access/login attempts, detected malicious code | ✅ `windows_eventlog` Security channel covers login success/failure on Windows HMIs/EWS; `journald`/`syslog_in` for Linux/relay devices; `linux_audit` (kernel audit trail: process executions, file-access watches, logins) for Linux BES Cyber Systems; **Sysmon preset** adds process/network telemetry where deployable | AV/EDR "malicious code detected" events arrive via their Event Log channels or syslog — the agent forwards them |
| **4.2** Alerting | Alert on detected security events (per capability) | ◐ Feedstock only — structured events with severity + fields | Alerting = logrok / SIEM rules |
| **4.3** Retention ≥ 90 days | Retain logs **on the applicable systems or BES Cyber Systems** ≥ 90 consecutive days | ◐ Retention itself is storage-side; the agent's **store-and-forward** prevents the classic violation path — logs lost during link outages before they ever reach the retention store | The "on-device 90 days" reading: keep Windows Event Log sizes adequate too (OS config, not agent) |
| **4.4** Review every 15 days | Review summarized/sampled logged events at least every 15 calendar days | ◐ Feedstock: complete delivery + structured fields make the SIEM-side summary review possible | Review process = yours |

**Why this agent fits OT specifically** (the constraint set, not just the checklist):
- **ESP / Purdue topology:** logs leave the electronic security perimeter through tightly controlled
  paths — **UDP "diode mode"** (one datagram/event + `sequence` SD for receiver-side gap detection)
  traverses unidirectional gateways; see [USER-GUIDE § Data diode](../USER-GUIDE.md#data-diode--one-way-link-ot-zones).
- **No call-home dependency:** runs fully unmanaged from a local config file (air-gapped mode); the
  management plane is optional and self-hosted.
- **Footprint:** single static cgo-free binary; no runtime, no installer dependencies — relevant for
  change-controlled HMI/EWS hosts.
