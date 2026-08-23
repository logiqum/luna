# NIS2 (Directive (EU) 2022/2555) — logging-relevant mapping

NIS2 applies to "essential" and "important" entities across 18 sectors (energy, transport, water, health,
digital infrastructure, manufacturing, …). Full member-state compliance enforcement ramps through **October
2026**. Penalties up to €10M / 2% global turnover (essential entities).
Sources: [EU digital strategy — NIS2](https://digital-strategy.ec.europa.eu/en/policies/nis2-directive) ·
[requirements overview (Dataguard)](https://www.dataguard.com/nis2/requirements/).
*Not legal advice — transposition details vary by member state.*

NIS2 doesn't prescribe an agent; it prescribes outcomes that are impossible without endpoint log
collection:

| Obligation | What it asks | Agent today | Notes / whose job |
|---|---|---|---|
| **Art. 21(2)** risk-management measures | Incident handling; security monitoring; policies on cryptography; supply-chain security | ✅ The collection layer: Windows (incl. Security channel + Sysmon), Linux journald, file, syslog relay — delivered with TLS/mTLS in transit | Risk analysis, policies, response = organizational |
| **Art. 23** incident reporting | **24h early warning → 72h notification → 1-month final report** for significant incidents | ◐ You can't reconstruct an incident timeline in 24h from logs that never left the affected boxes. The agent's prompt off-host forwarding + **no-gap store-and-forward** is what makes the 24/72h clock survivable | Detection, classification, reporting = SOC/platform |
| OT scope (new vs NIS1) | Manufacturing/OT entities are now in scope | ✅ The OT constraint set is the agent's home turf: air-gapped operation, diode-mode UDP egress, static binary for change-controlled hosts | See [nerc-cip.md](nerc-cip.md) § "Why this agent fits OT" |

**EU angle worth stating in procurement:** the agent is vendor-neutral (standard RFC 5424 syslog to any
aggregator — no foreign-cloud dependency, no forced telemetry) and runs fully self-hosted/on-prem,
including its optional management plane.
