# Migrating from Promtail

Promtail has been deprecated by Grafana: it is in long-term support until 28 February 2026 and reaches end of
life on 2 March 2026 ([Grafana's Promtail documentation](https://grafana.com/docs/loki/v3.6.x/send-data/promtail/stages/)
carries the notice and points to Grafana Alloy). The agent can take over a host's log collection from Promtail and
keep delivering to the same Loki, with the same stream labels. Or it can send the same data as RFC 5424 syslog to
logrok or any syslog receiver.

You don't rewrite the configuration by hand: the agent reads `config.yml`, converts what it can and writes an
agent configuration plus a report of everything it could not carry over.

## Is it a fit?

**Verified against:** the Grafana Loki v3.6.x documentation for Promtail (read 2026-10-16). Delivery into Loki 3.4.2 was run end to end; the converter was not run against Promtail itself.

| If your Promtail… | Then |
|---|---|
| tails files (`static_configs` with `__path__`), reads the systemd journal, listens for syslog or reads the Windows event log | converted (tables below) |
| pushes to Loki with `external_labels`, a tenant, basic auth, a bearer token or a CA | converted; credentials become placeholders |
| uses the `cri` / `docker` / `multiline` / `json` / `logfmt` / `labels` / `static_labels` / `drop` / `match` stages | converted where the effect can be reproduced exactly (see [Pipeline stages](#pipeline-stages)) |
| discovers pods with **`kubernetes_sd_configs`** | converted for node log collection (every container log file on the node); pod labels and annotations from the API server are **not available** |
| uses `regex`, `template`, `replace`, `output`, `timestamp`, `tenant`, `pack`, `geoip`, `limit`, `sampling` or `metrics` stages | **not converted**; each is reported with what to do instead |
| uses `docker_sd_configs`, `file_sd_configs`, Consul discovery, Kafka, GELF, Cloudflare, GCP, Azure or Heroku sources, or `loki_push_api` | **not converted** |
| authenticates to Loki with a **client certificate**, a proxy, OAuth 2 or custom headers | **not converted**; use basic auth or a bearer token, or terminate mTLS in a proxy |
| sets labels from the file name (the automatic `filename` label) on a **glob** path | the label is reproduced for a literal path only; the report says so |

## How it fits together

```mermaid
flowchart TB
  subgraph B["Before"]
    direction LR
    b_src["files / journal / syslog / event log"] --> b_tool["Promtail"]
    b_tool -->|"labels from targets + pipeline + external_labels"| b_dst["Loki"]
  end
  subgraph A["After"]
    direction LR
    a_src["files / journal / syslog / event log"] --> a_agent["agent"]
    a_agent -->|"the same labels"| a_dst["the same Loki"]
    a_agent -->|"RFC 5424 syslog over TLS"| a_alt["logrok / any syslog receiver"]
  end
  B ~~~ A
```

## Step 1 — convert

Get `luna-migrate` first: a separate download published with every agent release (`luna-migrate-<os>-<arch>`, saved as `luna-migrate` / `luna-migrate.exe`) — see [Getting luna-migrate](../USER-GUIDE.md#getting-luna-migrate).

Run the converter on the Promtail host (it only reads the configuration file):

```bash
luna-migrate promtail /etc/promtail/config.yml \
  -out /etc/logrok-universal-agent/agent.yaml \
  -report promtail-conversion.json
```

```powershell
& ".\luna-migrate.exe" promtail "C:\Program Files\Promtail\config.yml" `
  -out "C:\ProgramData\logrok-universal-agent\agent.yaml" `
  -report promtail-conversion.json
```

| Flag | Default | Meaning |
|---|---|---|
| `-out` | *(required)* | where to write the agent configuration; an existing file is never overwritten |
| `-report` | — | also write the full report as JSON |
| `-target` | `native` | `native`: send to the Loki servers Promtail sent to. `syslog`: everything to one RFC 5424 receiver over TLS |
| `-syslog-endpoint` | — | `host:port` of the syslog receiver for `-target syslog` |
| `-syslog-plain` | off | send the syslog output over plain TCP instead of TLS (a receiver without TLS, or a lab) |
| `-start-at` | `end` | where file inputs start on first run: `end` (only new lines: nothing Promtail already shipped is shipped again) or `beginning` |

Promtail's own positions file is not read: the agent keeps its own resume state. Promtail's
`-config.expand-env` placeholders (`${VAR}`, `${VAR:-default}`) are **not expanded**: the text is kept (in a client `url` each `${VAR}` becomes `REPLACE_WITH_<VAR>`), every use is
reported as `manual`, and the variable is listed under "fill in before deploying". luna-migrate checks the
configuration's structure, then runs the installed agent's `-check-config` (found beside `luna-migrate` or on the
`PATH`, or named with `-agent`), which builds every module and compiles every condition.

## Step 2 — read the report

| Severity | Meaning |
|---|---|
| `manual` | **not carried over**: a person must decide (each line says what to do instead) |
| `warning` | carried over, with a difference to check |
| `info` | carried over differently, nothing to do |
| `skipped` | deliberately not converted |

Each finding names the file it came from, and the line where the converter can tell (a few findings about the file as a whole have none). Then fill in what the report lists under "fill in before
deploying". Basic-auth users and passwords and bearer tokens are **never copied**, however they are written, and neither is a tenant written as `${VARIABLE}` (a literal tenant is copied): the generated
file says `REPLACE_WITH_…` (search for it). Credentials written into a URL (`https://user:pass@host`) are removed
from it.

## Labels: how the Loki streams stay the same

Loki labels are the data model downstream, so the conversion keeps them:

- every label a target sets (`static_configs.labels`, the journal, syslog and Windows `labels`) and every label from
  a `static_labels` stage becomes an **event field of the same name** on that input, and its name is listed in the
  Loki output's `label_fields`. An event that does not carry the field gets no such label, exactly as in Promtail;
- a literal `__path__` also sets the `filename` label, as Promtail does;
- a client's `external_labels` become that output's static `labels`; a label of the same name on the line wins, as in
  Promtail;
- a label taken from extracted data (`json` / `logfmt` + `labels`) is read per event: the value is parsed into a
  field prefixed with the job name (`pt_<job>_…`) and renamed to the label name (the rename is limited to that job; the parse itself runs on every input and adds prefixed fields to other inputs' lines too, which are not labels);
- internal labels (leading `__`) are dropped, as Promtail drops them after relabelling.

Keep the cardinality low: every distinct label value is a new stream, in Promtail and in the agent alike.

When there is more than one scrape config, a rule that must apply to one job only (a `drop`, a label rename) is
limited to it with a condition on the field `promtail.job`, which the converter stamps on that job's inputs. The one
exception is a `json` / `logfmt` parse stage: it becomes a `parse_json` / `parse_kv` processor that runs on every input
(it only adds prefixed fields), and the converter reports it.

## What is converted

### Inputs

| Promtail | Agent |
|---|---|
| `static_configs` with `__path__` | `filetail` `path`; `__path_exclude__` → `exclude`; labels → `add_fields`; resume state kept by the agent |
| `journal` | `journald`; labels → `add_fields`; `relabel_configs` copying `__journal__systemd_unit` (→ `unit`) or `__journal__hostname` (→ `host`) become labels |
| `syslog` (`listen_address`, `listen_protocol`) | `syslog_in` (`listen` / `protocols`); with `tls_config` → `tls_listen`, certificate, key and `tls_client_ca_file` |
| `windows_events` | `windows_eventlog`: `eventlog_name` → `channels`; an `xpath_query` or a `<QueryList>` → `query`; the agent keeps its own bookmark |
| `kubernetes_sd_configs` (role pod) | `filetail` over `/var/log/pods/*/*/*.log` (or the listed namespaces) in container-log mode with `add_k8s_metadata`; relabelling of namespace, pod and container names becomes labels |

### Pipeline stages

| Promtail | Agent |
|---|---|
| `cri` / `docker` | `filetail` `format: cri` / `docker` (container-log mode: reassembles split lines, takes the time from the log line) |
| `multiline` `firstline`, `max_wait_time` | `multiline.line_start_pattern`, `multiline.flush_timeout` (`max_lines` has no equivalent) |
| `json` `expressions`, `logfmt` `mapping` | a `parse_json` / `parse_kv` processor with a per-job field prefix, for plain dotted key paths; the log line itself is unchanged, as in Promtail |
| `labels`, `static_labels` | labels as described above |
| `drop` with `expression` (the line) or `source` + `value` / `expression` (an extracted value) | an `expr` drop rule limited to the job. A stage with `older_than`, `longer_than` or several sources is not converted at all |
| `match` | a selector over the job's static labels is decided during conversion (its nested stages are then used as written, or skipped); a line filter limits nested `drop` stages; a selector on a per-line label is `manual` |

A stage the converter cannot reproduce never leaves a later stage silently wrong: a `labels` or `drop` that
depends on data an unconverted stage produced is reported as `manual` too.

### Clients

| Promtail | Agent |
|---|---|
| each `clients[].url` | one `loki` output; every client receives every line, as in Promtail |
| `tenant_id` | `tenant` |
| `basic_auth`, `bearer_token` | `basic_auth_user` / `basic_auth_password`, `bearer_token` as **placeholders** |
| `tls_config` `ca_file`, `insecure_skip_verify` | `ca_file`, `insecure_skip_verify` |
| `external_labels` | the output's `labels` |
| `timeout` | `timeout` |
| `batchwait`, `batchsize`, `backoff_config` | not needed: the agent batches and retries from its disk spool |
| `-target syslog` | one `syslog` output over TLS; labels travel as structured-data fields, external labels as a global `add_fields` |

## What is not converted (and what to do)

| Promtail | Why / instead |
|---|---|
| `timestamp` stage | the agent stamps the time the line was read (in container mode, the time in the log line); check that Loki still accepts your entries in order |
| `regex`, `template`, `replace`, `output`, `pack`, `decolorize` | no equivalent processor; parse structured lines with `parse_json` / `parse_kv`, mask with `redact`, or work on the receiving side |
| `tenant` stage | a per-line tenant cannot be set; give each tenant its own output |
| `labeldrop`, `labelallow`, `structured_metadata` | remove the field from the input's `add_fields` / the output's `label_fields`; the output sends all remaining fields as structured metadata when `structured_metadata: true` |
| `limit`, `sampling`, `metrics`, `limits_config` | use the agent's reduction processors and limits if you relied on them |
| pod labels, annotations, `labelmap` | the agent does not call the Kubernetes API; only namespace, pod, container and node are available |
| `${VAR}` placeholders | kept as written and listed to fill in |
| client certificates, `proxy_url`, `oauth2`, `headers` on a client | not supported by the `loki` output |
| the automatic `filename` label on a glob | not reproduced; list each file as its own target if you query on it |

## Step 3 — cut over without a gap

Prefer a few seconds of duplicates to a gap:

1. Install the agent with the converted configuration and start it. With `start_at: end` it reads only lines
   written from now on.
2. Query Loki and confirm that entries arrive from the agent with your labels.
3. Stop and disable Promtail.

Between steps 1 and 3 both send new lines, so those few seconds arrive twice. Stopping Promtail *first* would lose
the lines written before the agent starts. The agent does not read Promtail's positions file.

## Sending to logrok instead

```bash
luna-migrate promtail /etc/promtail/config.yml -out agent.yaml \
  -target syslog -syslog-endpoint logrok.example.com:16575
```

Inputs and pipeline stages convert as above; only the destination changes. Add the TLS trust (`ca_file`, and a
client certificate if your receiver requires one) to the generated `syslog` output.

## Licensing

The converter is free. What the converted configuration uses follows the normal rules
([LICENSING](../LICENSING.md)): file tailing, journald, the syslog input, the parse and `expr` processors and one
RFC 5424 syslog destination are Core; the `loki` output, more than one destination, container-log mode, the Windows
Event Log input and the disk spool are Apex.
