# DORA (Regulation (EU) 2022/2554) — logging-relevant mapping

DORA applies to EU financial entities (banks, insurers, investment firms, crypto, …) and critical ICT
third-party providers, **in force since 2025-01-17** (regulation — uniform across member states, no
transposition). Source: [Sysdig — DORA & NIS2 compliance](https://sysdig.com/blog/the-first-cnapp-out-of-the-box-nis2-and-dora-compliance).
*Not legal advice; reporting deadlines below follow the implementing technical standards — confirm the
current ITS/RTS text with compliance counsel.*

| Obligation | What it asks | Agent today | Notes / whose job |
|---|---|---|---|
| ICT risk management (Ch. II) | Continuous monitoring of ICT systems; detection of anomalous activities | ✅ Endpoint log feedstock: Windows Security/Sysmon, Linux journald, app file logs — structured, UTC-stamped, delivered with TLS/mTLS | Detection/analytics = platform (logrok / SIEM) |
| **Major-incident reporting (Art. 19)** | Initial notification within **hours of classification** (4h per the reporting ITS, ≤24h from awareness), intermediate and final reports after | ◐ Same argument as NIS2 but tighter: an hours-scale clock requires logs to be **already centralized when the incident is classified**. Store-and-forward closes the outage gap; `/metrics` + heartbeat prove collection was actually running (no dark endpoints during the incident window) | Classification + reporting workflow = yours |
| Resilience testing (Ch. IV) | TLPT / scenario testing | ◐ Sysmon preset gives the red-team-visible telemetry (process creation w/ command lines, network connects, DNS) that makes purple-team exercises evidence-rich | Test program = yours |
| ICT third-party risk (Ch. V) | Oversight of ICT providers; exit strategies | ✅ Vendor-neutrality is the control: standard syslog to **any** aggregator means no collection-layer lock-in to one SIEM vendor — an exit-strategy talking point auditors accept | — |

**Honesty note:** the agent does not classify incidents, measure availability, or generate reports — it
guarantees the raw material (complete, timely, structured endpoint logs) exists where the 4/24/72-hour
clocks are answered.
