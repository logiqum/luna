# LUnA — the logrok universal agent

The cross-platform, vendor-neutral endpoint **log-forwarding agent**: a single static, dependency-free binary
that collects endpoint logs (Windows Event Log, ETW, WMI, files, journald, syslog, MQTT, HTTP, Linux audit,
macOS unified logs, …) and forwards them as standard **syslog (RFC 5424) over TCP/TLS** to any aggregator — a
[logrok](https://logrok.com) deployment by default, or any third-party syslog collector / SIEM. Disk
store-and-forward keeps data safe across air-gapped or intermittent links.

**This repository hosts the public downloads and the operator docs.** The friendly landing page is
**[logrok.com/download](https://logrok.com/download)**.

## Download

Every release attaches binaries, installers, and packages for all platforms under **stable, version-less
names** (so links never change). Grab them from the [**latest release**](https://github.com/logiqum/luna/releases/latest):

| Platform | Files |
|---|---|
| **Windows** x64 / arm64 | `luna-windows-amd64.msi` · `luna-windows-arm64.msi` (or the raw `.exe`) — Authenticode-signed |
| **Linux** x64 / arm64 / armv7 | `.deb` · `.rpm` · `.tar.gz` · or the raw static binary (`luna-linux-amd64`, …) |
| **macOS** Intel / Apple Silicon | `luna-darwin-amd64.pkg` · `luna-darwin-arm64.pkg` |
| **Container** (multi-arch) | `docker pull ghcr.io/logiqum/luna:latest` |
| **Other** | FreeBSD / NetBSD / OpenBSD / Solaris / AIX static binaries |

Quick starts:

```sh
# Debian/Ubuntu
sudo dpkg -i luna-linux-amd64.deb

# RHEL/SUSE
sudo rpm -i luna-linux-x86_64.rpm

# Windows (elevated)
msiexec /i luna-windows-amd64.msi /qn

# macOS
sudo installer -pkg luna-darwin-arm64.pkg -target /

# Container (gateway / Kubernetes)
docker run -d --name logrok-agent \
  -v /etc/logrok-agent/agent.yaml:/etc/logrok-agent/agent.yaml:ro \
  -v logrok-spool:/var/lib/logrok-agent \
  -p 5514:5514 \
  ghcr.io/logiqum/luna:latest
```

## Verify what you downloaded

On a security agent, don't skip this. Every release ships `SHA256SUMS`, and the container image is
cosign-signed.

```sh
# Checksums
curl -LO https://github.com/logiqum/luna/releases/latest/download/SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing

# Container signature (key-based cosign; public key lives in this repo)
cosign verify --key https://raw.githubusercontent.com/logiqum/luna/main/cosign.pub \
  ghcr.io/logiqum/luna:latest
```

## Documentation

- **[User guide](docs/USER-GUIDE.md)** — install, run, deploy, operate, troubleshoot, and platform notes.
- **[Configuration reference](docs/CONFIGURATION.md)** — every input, processor, and output with all options.

These docs are kept in sync with each release automatically.

## License

LUnA is licensed under the **[End-User License Agreement](LICENSE)**. The free **Core** capability set runs
with no license for non-commercial use, for use alongside a licensed logrok deployment, and for commercial
use on up to **10 agents**. The paid **Apex** capability set (Windows Event Log, disk store-and-forward,
central fleet management, and more) requires a license. Validation is fully offline — no phone-home.
