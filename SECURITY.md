# Security Policy

LUnA is a security-sensitive endpoint agent, and we take vulnerability reports seriously.

## Reporting a vulnerability

**Please do not open a public issue for security reports.** Use one of:

- **GitHub private vulnerability reporting** — the **Report a vulnerability** button on this repository's
  [Security tab](https://github.com/logiqum/luna/security/advisories/new) (preferred).
- **Email** — `contact@logiqum.com`

Please include the affected version, platform/OS, a description, and reproduction steps where possible. We aim
to acknowledge within **2 business days** and will keep you informed through investigation and remediation.
We're glad to credit reporters who would like to be credited.

## Supported versions

The **latest release** receives security fixes. Older releases are not maintained — please upgrade.

## Verifying your download

Every release ships a `SHA256SUMS` manifest, and the multi-arch container image is **cosign-signed**. Always
verify before deploying — see the README's *Verify what you downloaded* section. Public key: `cosign.pub`.

## Scope

This repository hosts the public downloads and operator documentation. Source-level matters are handled by
Logiqum; the channels above reach the right people regardless of where a fix ultimately lands.
