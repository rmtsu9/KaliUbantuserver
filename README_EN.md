# Kali–Ubuntu Security Monitoring Lab

## Purpose

This project is a controlled security-monitoring laboratory for generating and collecting web-traffic datasets. Kali Linux acts as the traffic-generation host, while Ubuntu Server hosts the intentionally vulnerable OWASP Juice Shop application and records evidence from multiple layers of the stack.

The lab is designed for learning, detection engineering, log analysis, and dataset preparation. It must only be used on systems and networks for which you have explicit authorization.

> **Ethics and scope:** OWASP Juice Shop is intentionally vulnerable and is intended for training and security-tool testing. Do not direct testing traffic at public systems or systems you do not own or have permission to assess.

## Learning Goals

- Understand how one web request appears at different observation points.
- Collect complementary evidence: raw packets, web-server logs, IDS events, and application logs.
- Build traceable datasets that can support incident analysis, rule tuning, or machine-learning experiments.
- Distinguish normal application activity from controlled, authorized security-test traffic.

## Lab Architecture

```text
┌──────────────────────────┐                           ┌──────────────────────────────────────────┐
│ Kali Linux               │                           │ Ubuntu Server                            │
│ 192.168.100.20           │                           │ 192.168.100.10                           │
│ eth1                     │                           │ enp0s8                                   │
│                          │                           │                                          │
│ Browser / test client    │────── HTTP traffic ──────▶│ Nginx reverse proxy                      │
└──────────────────────────┘                           │          │                               │
                                                       │          ▼                               │
                                                       │ OWASP Juice Shop (Docker, localhost:3000)│
                                                       │                                          │
                                                       │ Evidence collection:                     │
                                                       │ • tcpdump       → PCAP                   │
                                                       │ • Nginx         → access/error logs      │
                                                       │ • Suricata IDS  → EVE JSON               │
                                                       │ • Docker        → application log        │
                                                       └──────────────────────────────────────────┘
```

The lab network uses the following static addresses:

| Host | Interface | Address | Role |
| --- | --- | --- | --- |
| Ubuntu Server | `enp0s8` | `192.168.100.10/24` | Web server, reverse proxy, IDS, packet capture |
| Kali Linux | `eth1` | `192.168.100.20/24` | Browser and authorized test-traffic generator |

## End-to-End Request Flow

When a client on Kali requests `http://192.168.100.10`, the request travels through the following path:

1. Kali sends TCP/IP packets to Ubuntu through the isolated lab interface.
2. `tcpdump` can copy those packets from `enp0s8` into a PCAP file.
3. Suricata observes the traffic, decodes supported protocols, and evaluates it against its configured rules.
4. Nginx accepts the HTTP request, writes a web-server log entry, and proxies the request to the local Juice Shop service.
5. Juice Shop processes the request inside its Docker container and may write application output to `stdout` or `stderr`.
6. Nginx returns the upstream response to Kali.

This path makes the same activity observable at several layers. An analyst can start with an IDS alert, locate the matching request in Nginx logs, inspect application behavior in container logs, and use the PCAP to verify the underlying network exchange.

## Core Components and Theory

### Kali Linux: the client-side observation point

Kali is the client host in this design. Its purpose is to generate known, labelled traffic in the isolated network. Some traffic can represent routine user behavior, such as browsing pages or submitting ordinary forms. Other traffic can represent authorized security tests against the local Juice Shop instance.

Keeping the traffic generator on a separate host matters because source IP address, network direction, timing, and connection behavior become visible to the monitoring host. These properties are useful when correlating events and later assigning dataset labels.

### Nginx: reverse proxy and HTTP evidence source

Nginx is positioned at the HTTP entry point. A reverse proxy accepts a request on behalf of an upstream application and forwards it internally. In this lab, Nginx receives requests on port 80 and forwards them to Juice Shop at `127.0.0.1:3000` through the `proxy_pass` directive.

Nginx is valuable for dataset collection because its logs operate at the HTTP transaction layer rather than the packet layer. A typical access-log record can include:

- client IP address;
- timestamp;
- HTTP method and requested path;
- response status code;
- response size; and
- user-agent or referrer, if configured in the log format.

The access log answers questions such as *which client requested which resource and what did the server return?* The error log supplements it with proxy failures, upstream errors, and server-side issues.

Nginx alone is **not** a web application firewall (WAF). It becomes part of a WAF solution only when an appropriate security module and ruleset are installed and configured. The basic Nginx setup in this repository should therefore be understood as a reverse-proxy and logging layer.

Reference: [NGINX HTTP Proxy Module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)

### OWASP Juice Shop: a safe target for security training

OWASP Juice Shop is an intentionally insecure web application created for security training, demonstrations, capture-the-flag exercises, and testing security tools. It includes a broad range of vulnerability categories, which makes it suitable for controlled detection experiments.

The application is run as a Docker container. Containerization packages the application and its dependencies together, simplifying deployment and making the lab easier to rebuild. It does **not** by itself make a vulnerable application safe to expose; the container should remain accessible only within the authorized lab boundary.

Reference: [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/)

### Docker logs: application-layer context

Docker collects a container's standard output and standard error according to the configured logging driver. The `docker logs juiceshop` command retrieves the logs currently held for the named container.

These logs can contain application startup messages and request-processing context that may not be available in Nginx or Suricata. They are therefore useful for corroboration, although their exact content depends on the application's logging behavior and Docker logging configuration.

Reference: [Docker — `docker container logs`](https://docs.docker.com/reference/cli/docker/container/logs/)

### Suricata: signature-based network intrusion detection

Suricata is a network intrusion detection system (IDS). It observes traffic from a configured interface, decodes protocols, and evaluates traffic against signatures—also called rules. A rule can inspect characteristics such as protocol fields, addresses, ports, HTTP metadata, or content. When traffic satisfies a rule's conditions, Suricata can produce an alert.

An IDS alert is evidence that a rule matched; it is not, by itself, definitive proof that an attack succeeded. False positives can occur when benign traffic resembles a rule pattern, while false negatives can occur when traffic is encrypted, fragmented, evasive, unsupported, or simply not covered by the active rules. This is why correlation with web, application, and packet evidence is essential.

Suricata writes structured events to `eve.json` when EVE JSON output is enabled. Depending on configuration, these events may include alerts, HTTP metadata, DNS records, flow information, TLS records, file information, and statistics. The event types present in a dataset depend on `suricata.yaml`, the capture configuration, and the enabled ruleset.

Reference: [Suricata Quickstart Guide](https://docs.suricata.io/en/latest/quickstart.html)

### tcpdump and PCAP: packet-level ground evidence

`tcpdump` records traffic observed on a network interface. The command used in this lab writes packets captured on `enp0s8` to `attack_traffic.pcap`.

PCAP is a packet-capture format commonly used for network forensics and protocol analysis. It preserves the sequence and timing of captured frames, allowing analysts to examine IP addresses, ports, TCP sessions, DNS messages, HTTP exchanges, and other protocol fields. The available payload detail depends on the capture point and on whether traffic is encrypted.

Packet capture has both strengths and limitations:

| Strength | Limitation |
| --- | --- |
| Preserves low-level communication details | Files can grow quickly and require storage planning |
| Supports independent re-analysis with other tools | Encrypted traffic limits application-payload visibility |
| Helps verify whether a connection or request was actually transmitted | A capture can miss traffic if started late, stopped early, or overloaded |

Stopping `tcpdump` with `Ctrl+C` is important: it lets the program close the output file cleanly before the PCAP is archived.

## Why Multi-Layer Logging Matters

No single source fully describes a security event. Each source answers different questions.

| Evidence source | Primary perspective | Useful questions |
| --- | --- | --- |
| PCAP | Network packets and session sequence | Was traffic transmitted? Which ports, flags, payloads, and timing were observed? |
| Nginx access log | HTTP request/response transaction | Which URI was requested? Which status code was returned? |
| Nginx error log | Proxy and web-server failures | Did an upstream or web-server error occur? |
| Suricata EVE JSON | IDS detection and protocol metadata | Did an active detection rule match? What metadata accompanied it? |
| Juice Shop application log | Application execution | What did the application report while handling the request? |

This approach is known broadly as **defense in depth** for telemetry: visibility is intentionally distributed across layers so that a blind spot in one source can be checked against another. OWASP similarly emphasizes application logging as a security capability that supports monitoring, investigation, and incident response.

Reference: [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)

## Dataset Design Principles

### 1. Define the unit of analysis

Before collecting data, decide what one record represents. Common choices include:

- one packet;
- one network flow;
- one HTTP request;
- one Suricata alert; or
- one time window containing features from several sources.

The correct choice depends on the analysis goal. Packet-level models capture network detail but can be large and noisy. Request-level records are often easier to interpret for web analysis. Flow- or window-level records are useful when behavior unfolds over multiple packets or requests.

### 2. Preserve labels and metadata

For supervised analysis, every collection run should have an external experiment note describing at least:

| Field | Example |
| --- | --- |
| Run ID | `baseline-001` or `authorized-test-003` |
| Start and end time | ISO 8601 with timezone |
| Traffic class | normal / authorized test / scan |
| Source and destination | `192.168.100.20` → `192.168.100.10` |
| Active Suricata ruleset | version or update timestamp |
| Relevant configuration | Nginx, Suricata, Docker image version |
| Operator and authorization reference | accountable owner and lab approval |

Avoid relying only on filenames to describe labels. Keep a small manifest file or experiment log alongside the exported archive.

### 3. Maintain time consistency

Correlation depends on time. Kali and Ubuntu should use the same timezone or, preferably, synchronized time. If the clocks differ, an IDS alert, an Nginx request, and a PCAP packet can appear to refer to different events even when they are the same activity.

Use explicit timestamps and record the timezone in experiment notes. When normalizing data for analysis, convert timestamps to a single standard such as UTC while preserving the original timestamp where needed.

### 4. Separate baseline and test traffic

Collect normal traffic in separate, clearly labelled runs before collecting authorized security-test traffic. Baseline traffic might include browsing public pages, product searches, logins with designated test accounts, and normal form submissions.

This separation reduces label ambiguity. A mixed capture without a trustworthy timeline is difficult to use for detection evaluation because it is unclear which requests should be regarded as normal or suspicious.

### 5. Understand capture completeness

Every dataset is an observation, not an absolute record of reality. Completeness can be affected by:

- capture started after some activity already occurred;
- capture stopped before connections completed;
- interface mismatch;
- VM resource pressure or packet drops;
- log rotation or overwritten files;
- missing Suricata rules; and
- differences in time settings.

Record these limitations in the dataset manifest. Transparent limitations make the resulting dataset more credible and easier to reproduce.

## Collection Workflow

### Before collection

Follow [Start.md](Start.md) to:

1. assign the lab IP addresses;
2. verify connectivity from Kali to Ubuntu;
3. start Juice Shop, Nginx, and Suricata;
4. verify the application locally and from Kali; and
5. start packet capture before generating traffic.

### During collection

- Keep a timestamped activity log for each traffic scenario.
- Do not change system configuration or Suricata rules in the middle of a labelled run unless the change is recorded.
- Watch free disk space, especially while recording PCAP files.
- Keep test traffic inside the authorized lab scope.

### After collection

Follow [End.md](End.md) to:

1. stop `tcpdump` cleanly;
2. gather Nginx, Suricata, PCAP, and Juice Shop logs;
3. assign usable file ownership;
4. create a timestamped `.tar.gz` archive; and
5. export the archive to the Windows host through WinSCP.

## Output Dataset Structure

```text
dataset_YYYY-MM-DD_HHMM.tar.gz
└── my_dataset/
    ├── attack_traffic.pcap       # Raw network packet capture, when present
    ├── nginx_access.log          # HTTP request/response records
    ├── nginx_error.log           # Nginx and upstream error records
    ├── suricata_eve.json         # IDS alerts and structured network events
    └── juiceshop_app.log         # Container/application output
```

## Operational Notes

- `Start.md` uses temporary `ip addr add` commands. These addresses do not persist after a reboot; this is intentional for the stated lab workflow.
- Docker publishes Juice Shop only on `127.0.0.1:3000`; Nginx exposes the application to the lab network on port 80.
- Ensure Suricata's monitored interface is `enp0s8` and its `HOME_NET` includes `192.168.100.0/24`.
- The exported archive can contain sensitive log data. Store it securely and do not publish captures containing credentials, tokens, or personal data.

## Related Documentation

- [Requirements and installation](requirement.md)
- [Start-of-collection checklist](Start.md)
- [End-of-collection checklist](End.md)

## References

1. [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/)
2. [Suricata Documentation — Quickstart Guide](https://docs.suricata.io/en/latest/quickstart.html)
3. [NGINX Documentation — HTTP Proxy Module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
4. [Docker Documentation — `docker container logs`](https://docs.docker.com/reference/cli/docker/container/logs/)
5. [OWASP Cheat Sheet Series — Logging](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
