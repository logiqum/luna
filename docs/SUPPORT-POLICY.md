# LUnA — Support, Versioning & Deprecation Policy

This document states which releases of LUnA (Logrok Universal Agent) receive fixes, how versions
are numbered, and how much notice you get before an operator-facing surface changes.

## Versioning

Releases follow [Semantic Versioning](https://semver.org/) (`MAJOR.MINOR.PATCH`):

- **PATCH** — fixes only; no configuration or behavior changes required on your side.
- **MINOR** — new capability, backward compatible. Existing configurations keep working. One exception:
  licensing enforcement — a capability used outside your entitlement may be withdrawn in any release; the
  compatibility promise covers the configuration, CLI and module surfaces you are licensed for.
- **MAJOR** — may contain breaking changes. Every breaking change is listed in the release notes
  with a migration note.

## Support window

**The latest release line is the supported release.** Bug fixes and security fixes ship in the
next release on the current line; the supported remediation path for an issue in an older version
is upgrading to the latest release. We do not maintain long-term-support branches or backport
fixes to past minor versions.

What this means in practice:

- Report issues on any version — but reproduction and fixes are verified against the latest
  release, and the fix ships there.
- Security vulnerabilities are handled per our [Security Policy](../SECURITY.md), and a fix that
  warrants it ships as an expedited patch release on the current line.
- Upgrades are designed to be drop-in within a major version: configuration files, disk-buffer
  contents, and checkpoints carry forward without manual migration.

## Deprecation policy

When an operator-facing surface (a configuration key, a module type, a CLI flag, a default) is
scheduled to change or be removed:

- The deprecation is **announced in the release notes at least one minor version before** any
  behavior changes. Where practical, the agent also logs a deprecation warning when the
  deprecated surface is used.
- The deprecated surface **keeps working until the next major version**. Removal or a breaking
  change of a documented surface only happens at a major version boundary, consistent with
  Semantic Versioning.
- Every deprecation notice names the replacement (or states that there is none) so a
  configuration can be migrated ahead of time.

## Configuration compatibility

A configuration written for one version keeps working on the next. Precisely:

- **Agent X.Y accepts, unchanged, every configuration that agent X.(Y-1) accepted.** Every key,
  module type and option of the previous minor version is still recognised — honoured, or
  deprecated with a warning — never refused and never silently ignored.
- **Agent X.0 accepts every configuration that any X-1 release accepted.** A major version is the
  only place a surface is removed, and it still accepts the whole previous major's configurations
  on the way in (the deprecation policy above tells you in advance which keys will stop working
  *after* the upgrade).
- A key or module can therefore only be removed one full minor version after its deprecation was
  announced, and only at a major version boundary.

This is what makes upgrading from the control plane safe: before a new binary is installed it is
run once against the host's live configuration and must accept it. The rule is enforced
mechanically before every release — the new binary must accept every reference configuration the
last release and the previous minor version shipped — and it determines the oldest version a
release pack is offered to (`min_upgrade_from`: X.(Y-1).0 for X.Y, (X-1).0.0 for X.0). To check
a configuration yourself against a binary: `logrok-universal-agent -check-config -config <file>`
prints a one-line verdict naming any key, module or module option the binary does not know, or any
value a module refuses.

## Platform coverage

Platform support tiers (which OS/architecture builds are runtime-verified versus
cross-compiled) are documented in the [User Guide](USER-GUIDE.md#every-platform-we-ship-and-how-far-each-is-verified). The
support window above applies uniformly: fixes for any supported platform ship on the latest
release line.

## Software bill of materials

Every release ships a [CycloneDX](https://cyclonedx.org/) SBOM (`luna-sbom.cdx.json` among the
release assets) listing the release's third-party components and their licenses, alongside the
`NOTICE` attribution file.
