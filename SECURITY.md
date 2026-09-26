# Security Policy

## Reporting a vulnerability

Email **security@logiqum.com**. Please do not open a public issue for anything you believe is a
security vulnerability.

Include what you can: the agent version (`logrok-universal-agent -version`), platform (OS/arch),
deployment mode (standalone or centrally managed), steps to reproduce, and the impact as you
understand it.

What to expect:

- **Acknowledgement within 3 business days.**
- Triage and a severity assessment, shared with you.
- A fix targeted at the latest release; critical issues may warrant an out-of-band release.
- **Coordinated disclosure**: we ask for up to 90 days before public disclosure, and we credit
  reporters in the release notes unless you prefer otherwise.

There is currently no paid bug-bounty program.

## Scope

- The logrok universal agent: the agent binary and every packaged artifact we publish for it
  (installers, service wrappers, container image, presets and example configurations).
- Vulnerabilities in third-party dependencies are in scope when the agent's usage makes them
  exploitable. Every release runs a reachability-based vulnerability gate against pinned
  dependencies, and assessments of non-exploitable findings are published in
  [docs/security/cve-assessments.md](docs/security/cve-assessments.md).

## Supported versions

Security fixes land in the **latest release**. The agent upgrades in place from any prior 1.x
version (configuration and on-disk state are forward-compatible within a major version), so staying
current is the supported posture.

## Verifying what you run

Release artifacts ship with SHA-256 checksums; binaries and the container image are signed —
verification instructions and the public key are published alongside the release artifacts. The
agent is a single static binary with no runtime dependencies, which keeps the surface auditable:
what you download is what runs.
