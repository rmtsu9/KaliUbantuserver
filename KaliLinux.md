# Kali Linux Toolkit for Controlled Web-Security Dataset Collection

## 1. Purpose and Safe Scope

This guide explains how Kali Linux can be used as a **controlled traffic-generation host** in the Kali–Ubuntu Security Monitoring Lab. Its aim is to create understandable, repeatable datasets while observing how the Ubuntu Server records the same activity through:

- Logstash JSONL events collected through Filebeat and Packetbeat; and
- optional packet capture (`tcpdump` / PCAP).
- Nginx access and error logs;
- Suricata IDS events (`eve.json`); and
- OWASP Juice Shop application logs.

All activity described here must remain within the authorized lab target:

```text
Kali Linux:    192.168.100.20
Ubuntu target: 192.168.100.10
Web target:    http://192.168.100.10
```

> Do not substitute a public IP address, a workplace system, or any system outside your written authorization. This guide is for the local OWASP Juice Shop lab only.

Use purpose-created test accounts only. Do not retrieve data beyond the intended training scenario, open database/operating-system shells, alter application data, use external callbacks, or place real credentials, cookies, tokens, or personal data in published Dataset files.

## Pipeline-Specific Collection Note

The primary Dataset artifact for the current lab architecture is:

```text
/var/log/logstash/ai_dataset_YYYY_MM_DD.jsonl
```

Filebeat forwards configured logs, Packetbeat produces network events, and Logstash writes the resulting JSONL file as background services. The standard collection workflow therefore starts **Logstash, Filebeat, and Packetbeat**; it does not require a foreground `tcpdump` terminal. Every PCAP reference below refers to optional supplemental evidence only.

## 2. A Necessary Correction: Tools Do Not “Cover OWASP Top 10 Completely”

Kali contains many useful tools, but no collection of scanners can prove that an application is free of every OWASP Top 10 risk. Automated tools are strongest at discovering observable technical patterns, such as exposed services, unsafe headers, common injection indicators, or known misconfigurations.

Several OWASP categories require additional evidence that a network scanner cannot see:

- **Insecure Design** requires understanding intended business workflows and threat models.
- **Software Supply Chain Failures** require dependency, build-pipeline, and provenance review.
- **Software and Data Integrity Failures** often require CI/CD, signing, update, and architecture review.
- **Logging and Alerting Failures** require validating that expected events are collected, retained, and actionable.

The current [OWASP Top 10:2025](https://owasp.org/Top10/) should be treated as a risk-awareness framework—not as a checklist that one tool can automatically complete.

## 3. Why Use Kali Linux in This Lab?

Kali is not merely an “attack machine.” In a dataset lab, it is a **controlled experiment client**. It produces traffic whose tool, settings, timing, target, and intended label are known in advance.

That controlled origin makes later analysis more reliable. For example, if a low-rate directory discovery run begins at 14:10, the corresponding burst of HTTP `404` responses in Nginx and related Suricata events can be linked to that run. Without this experiment context, similar log records are much harder to label correctly.

### The observation model

```text
Kali tool action
      │
      ├── Pipeline view: normalized Filebeat and Packetbeat events in Logstash JSONL
      ├── IDS view: protocol metadata and rule matches in Suricata EVE JSON
      ├── Web view: method, path, status, bytes, client IP in Nginx logs
      └── Application view: Juice Shop container output
```

Each layer tells only part of the story. A Suricata alert is a rule match, not automatic proof of a successful compromise. Likewise, an HTTP `200` response does not necessarily mean that a security test succeeded. Cross-checking multiple sources is the central idea behind this lab.

## 4. Prerequisites on Kali Linux

Before installing tools, complete the network and service prerequisites in [requirement.md](requirement.md), then follow [Start.md](Start.md) before each collection run.

### Core packages

Run the following only on the Kali Linux VM:

```bash
sudo apt update
sudo apt install -y \
  nmap nikto whatweb ffuf gobuster feroxbuster \
  sqlmap hydra curl httpie jq \
  apache2-utils wrk tshark wireshark
```

Optional GUI tools can be installed when needed:

```bash
sudo apt install -y burpsuite zaproxy
```

Package availability can differ between Kali releases. Check before installing an optional tool:

```bash
apt search <package-name>
apt show <package-name>
```

Kali tool availability depends on the installed metapackages; do not assume every security tool is present in every installation. See [Kali Linux metapackages](https://www.kali.org/docs/general-use/metapackages/).

Use Kali's own repositories; do not mix Ubuntu or Debian repositories into Kali. Kali explicitly warns that mixing operating-system repositories can break the installation. See [Kali package repository guidance](https://www.kali.org/docs/general-use/kali-apt-sources/).

### Verify the lab before generating traffic

```bash
export LAB_HOST="192.168.100.10"
export LAB_URL="http://192.168.100.10"

ping -c 4 "$LAB_HOST"
curl -I "$LAB_URL"
```

Expected result: `0% packet loss` from `ping` and an HTTP success response from `curl`.

## 5. Tool Groups and Their Dataset Value

| Group | Representative tools | Primary purpose | Expected evidence on Ubuntu |
| --- | --- | --- | --- |
| Network discovery | Nmap | Identify reachable ports and services | Logstash JSONL; possibly Suricata scan alerts; optional PCAP |
| Web fingerprinting | WhatWeb, Nikto | Inspect web technologies and common configuration issues | Nginx requests, Logstash JSONL, IDS alerts when rules match |
| Content discovery | ffuf, Gobuster, Feroxbuster | Discover reachable paths using a bounded wordlist | High volume of HTTP paths and status codes |
| Intercepting proxies / DAST | Burp Suite, OWASP ZAP | Inspect, replay, and validate HTTP requests inside defined scope | Precise request sequences in Nginx and Logstash JSONL |
| Input-validation testing | sqlmap, Burp Repeater, ZAP | Validate controlled input-handling scenarios | HTTP parameter patterns; possible IDS alerts |
| Authentication / authorization | Burp Repeater, ZAP Fuzzer, Hydra | Test designated accounts and access-control decisions | Login events, `401`/`403`/`200` patterns, alert telemetry |
| API investigation | curl, HTTPie, jq, Arjun* | Understand REST endpoints and JSON responses | API request/response sequences |
| Bounded resilience tests | ApacheBench (`ab`), `wrk` | Compare normal load with a small, defined request burst | Flow volume, Nginx rates/statuses, resource telemetry |
| Local analysis | Wireshark, TShark, jq | Inspect JSONL and optional captured PCAP evidence | No new target traffic unless capture is started locally |

\*Arjun is optional and may not be packaged in every Kali release.

## 6. Reconnaissance and Scanning

Reconnaissance is the process of learning what is exposed before performing any deeper test. In this lab, reconnaissance is useful because it creates recognizable traffic patterns that can be compared with ordinary browsing.

### 6.1 Nmap — network and service discovery

Nmap uses network probes to determine host and port state. Service/version detection (`-sV`) sends service-aware probes to open ports and matches responses against Nmap's detection database. This provides more evidence than assuming a service solely from its port number.

Reference: [Nmap service and version detection](https://nmap.org/book/vscan.html)

Use a small, low-speed scan against the **single lab host**:

```bash
mkdir -p ~/lab-results
nmap -sV --version-light -T2 -p 80 -oA ~/lab-results/nmap-http "$LAB_HOST"
```

What it creates:

- normalized events in the Logstash JSONL file, plus optional PCAP evidence when enabled;
- a service-discovery record in `~/lab-results/`; and
- potentially a scan-related Suricata event, depending on active rules.

Why this profile is bounded:

- `-p 80` limits the scan to the lab web service;
- `--version-light` lowers the number of version-detection probes;
- `-T2` uses a conservative timing profile; and
- `-oA` preserves normal, XML, and grepable results for later comparison.

Avoid broad, high-speed scans such as a full-port scan with maximum timing while collecting a clean web dataset. They can dominate the traffic mix and make labels less useful.

### 6.2 WhatWeb — web technology fingerprinting

WhatWeb fingerprints visible web technologies from HTTP behavior, headers, cookies, and page content. It is useful for generating a short sequence of ordinary-looking HTTP requests while documenting server-side characteristics.

```bash
whatweb --log-verbose=~/lab-results/whatweb.log "$LAB_URL"
```

Expected artifacts: Nginx access entries, Logstash JSONL events, and a local WhatWeb result log; optional PCAP traffic is available only when `tcpdump` was enabled.

### 6.3 Nikto — web-server checks

Nikto sends a catalogue of web requests that can reveal common web-server issues, exposed paths, and configuration weaknesses. It is useful as a **scanner-traffic class** in a dataset, but its findings must be verified manually. A finding can be outdated, context-dependent, or a false positive.

```bash
nikto -h "$LAB_URL" -output ~/lab-results/nikto.txt -Format txt
```

Expected artifacts:

- a diverse series of Nginx requests, often including `404`, `301`, `403`, and `200` responses;
- packet-level HTTP exchanges; and
- possible Suricata signature matches, depending on the ruleset.

### 6.4 Content discovery — ffuf, Gobuster, and Feroxbuster

Content discovery sends candidate path names from a wordlist. This is commonly called directory or content enumeration. It does not prove that a discovered path is sensitive; it only shows that the server responded differently from a missing path.

Use **one** tool per run and a small wordlist. Example with ffuf:

```bash
ffuf \
  -u "$LAB_URL/FUZZ" \
  -w /usr/share/wordlists/dirb/common.txt \
  -rate 10 \
  -of json \
  -o ~/lab-results/ffuf-common.json
```

The `-rate 10` limit is intentional. It creates a clear enumeration pattern without needlessly exhausting the lab service. A useful label for this run is `content-discovery-low-rate`.

Alternatives:

| Tool | When to choose it |
| --- | --- |
| `ffuf` | Rate control and structured output are important |
| `gobuster` | A simple, focused command-line wordlist run is sufficient |
| `feroxbuster` | Recursive content discovery is needed; keep recursion and rate limits small |

## 7. Web-Application Testing Tools

### 7.1 Burp Suite — manual request analysis and validation

Burp Suite is most useful in this lab as an intercepting proxy and manual validation environment. Configure its **Target Scope** so only `http://192.168.100.10` is in scope. Then browse Juice Shop through the proxy and send selected requests to Repeater.

Burp Repeater lets a tester modify and resend individual HTTP or WebSocket messages. It is especially valuable for controlled comparisons: alter one parameter, resend the request, then compare the status code, response body, timing, and logs.

Reference: [PortSwigger — Burp Repeater](https://portswigger.net/burp/documentation/desktop/tools/repeater)

For dataset quality, record the original request and each intentional variation. Avoid large unlabelled Intruder runs; they create high-volume traffic without clear ground truth.

### 7.2 OWASP ZAP — dynamic application security testing (DAST)

OWASP ZAP can crawl an application and perform active or passive checks. Its key theoretical distinction is:

- **Passive scanning** observes messages that already pass through ZAP and does not alter them.
- **Active scanning** sends additional requests and variations to test behavior.

Use passive scanning while building a normal-traffic baseline. Use active scanning only against the authorized Juice Shop target and only in a separately labelled experiment window. DAST findings are hypotheses that require confirmation; they are not proof of exploitability.

ZAP explicitly treats active scanning as an attack and notes that automated active scans cannot identify logical problems such as broken access control. Scope, policy, duration, and concurrency must therefore be limited before running it.

Reference: [OWASP ZAP documentation](https://www.zaproxy.org/docs/) and [ZAP Active Scan](https://www.zaproxy.org/docs/desktop/start/features/ascan/)

### 7.3 sqlmap — controlled SQL-injection validation

SQL injection is one member of the broader OWASP **Injection** category. SQLmap automates testing of parameters for SQL-injection behavior. It can generate a substantial number of requests, so use the smallest safe profile and record the exact command and time window.

Example for the locally authorized Juice Shop endpoint:

```bash
sqlmap \
  -u "$LAB_URL/rest/products/search?q=test" \
  --batch \
  --level=1 \
  --risk=1 \
  --threads=1 \
  --output-dir=~/lab-results/sqlmap
```

Important interpretation notes:

- A scanner response is not sufficient evidence by itself; correlate it with the application's intended training scenario and server-side logs.
- Do not use destructive, data-modifying, shell, or operating-system takeover options when collecting a baseline dataset.
- Keep SQLi traffic in a separate run from normal browsing so it can be labelled clearly.

Reference: [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)

### 7.4 Cross-site scripting (XSS) — theory-first validation

XSS occurs when untrusted input is treated as executable client-side content in a browser context. It has several forms—including reflected, stored, and DOM-based XSS—and each may appear differently in web logs and IDS data.

Tools such as Burp Repeater, ZAP, and XSStrike can help create and compare input-validation cases. For a training dataset, prefer a documented Juice Shop exercise or a benign, unique test marker rather than copying untracked payloads from the internet. Record:

- input location (query parameter, JSON field, form field, or header);
- request timestamp and source IP;
- whether the response reflected or stored the marker;
- browser-side observation; and
- matching Nginx, Suricata, PCAP, and application events.

Reference: [OWASP Cross Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)

### 7.5 Command injection and path traversal — use controlled markers

Command injection and path traversal can have harmful side effects even in a lab. The best dataset practice is to validate a known Juice Shop training scenario using an **inert, pre-created test marker** rather than attempting to read sensitive operating-system files or execute arbitrary commands.

Tools commonly associated with these categories include `commix` (command-injection testing) and content/request tools such as Burp Repeater or `curl` (path traversal validation). Do not run automated command-execution or bulk file-extraction modes. Focus on detecting the request pattern and verifying its expected, limited behavior.

## 8. Authentication, Session, and Access-Control Testing

### 8.1 Theory: authentication is not authorization

- **Authentication** answers: *Who is this user?*
- **Authorization** answers: *May this authenticated user access this object or action?*

An application can authenticate a user correctly and still fail authorization. For example, an insecure direct object reference (IDOR) occurs when user-controlled input directly selects an object without an adequate authorization check. This can result in horizontal or vertical privilege escalation.

Reference: [PortSwigger — IDOR](https://portswigger.net/web-security/access-control/idor)

### 8.2 Safe test design for designated lab accounts

Create only purpose-made test accounts. For each scenario, use a small number of known attempts and label the expected result. Example categories:

| Scenario | Expected outcome | Useful evidence |
| --- | --- | --- |
| Valid login | Session is established | Nginx status, container event, cookie/session behavior |
| Invalid login | Authentication is rejected | Repeated request pattern, status/error response, IDS telemetry |
| Same-role object request | Access is allowed only to the correct test object | Request path/parameter and returned authorization decision |
| Cross-user object request | Access is denied | `403` or application-specific denial behavior; server logs |

### 8.3 Hydra and automated credential guessing

Hydra can automate authentication attempts. In a production environment, this can lock accounts, overwhelm authentication systems, and violate authorization. In this lab, use it only if your experiment explicitly requires a **small, rate-limited, purpose-created credential set**.

For most datasets, a proxy-based, manually controlled set of valid and invalid login requests is more useful because every attempt has an unambiguous label. Do not use leaked password lists, real credentials, or unbounded brute-force settings.

### 8.4 JWT and session-token analysis

JSON Web Tokens (JWTs) and cookies are application-session artifacts, not proof of authorization. A secure test evaluates whether the server validates a token's signature, expiry, audience, and authorization claims—rather than trusting client-controlled data.

Burp Repeater is well suited to controlled, one-variable-at-a-time comparisons. Record token format and metadata only when allowed; never publish real secrets, session IDs, or personal data in a dataset.

## 9. API and Parameter Discovery

Modern web applications often use REST or JSON APIs. Juice Shop contains API-style endpoints, so API traffic should be collected as a distinct data class.

### curl, HTTPie, and jq

These tools are useful for producing simple, reproducible API requests and inspecting JSON locally:

```bash
curl -sS "$LAB_URL/rest/products/search?q=apple" | jq .
http GET "$LAB_URL/rest/products/search" q==apple
```

Use a unique, harmless search term for each run, such as `dataset-baseline-001`, so that logs can be correlated by time and query value. Avoid sending tokens or credentials in command history; use a purpose-created test account and an approved secret-handling method when authentication is needed.

### Arjun and parameter discovery

Arjun is a parameter-discovery tool. It can be useful for observing how an application responds when likely parameter names are supplied, but it should run with a small dictionary and restricted rate in an authorized lab. Treat its results as candidates for manual validation, not as confirmed vulnerabilities.

## 10. Bounded Load and Resilience Experiments

### Why `ab` or `wrk` are better than raw packet floods for this lab

For a web-application dataset, HTTP-aware load tools produce requests that Nginx and the application can log meaningfully. By contrast, a raw packet tool such as `hping3` primarily tests network-stack behavior; it may not create HTTP application events at all.

Use a small, explicit request count and concurrency value. Example:

```bash
ab -n 50 -c 5 "$LAB_URL/"
```

This means **50 total HTTP requests** with no more than **5 concurrent requests**. It is a bounded measurement, not a denial-of-service exercise.

Compare it with a baseline:

```bash
for i in {1..10}; do
  curl -sS -o /dev/null -w '%{http_code} %{time_total}\n' "$LAB_URL/"
  sleep 1
done
```

Record the two runs separately. Useful comparison features include request rate, response-time distribution, Nginx status-code counts, Suricata flow counts, packet count, and any application errors.

Never increase request count or concurrency until you have defined a stop condition, checked available VM resources, and obtained authorization for the experiment. Stop immediately if the lab becomes unavailable or response errors rise unexpectedly.

## 11. Mapping Tool Behavior to OWASP Top 10:2025

The table below shows where the tools can provide **evidence or test traffic**. It does not claim complete automated coverage.

| OWASP category | Useful Kali activity | What the tool cannot establish alone |
| --- | --- | --- |
| A01 Broken Access Control | Burp Repeater; controlled cross-user object tests | Whether access rules match the complete business policy |
| A02 Security Misconfiguration | Nmap, WhatWeb, Nikto, ZAP passive checks | Whether every deployment/configuration layer is secure |
| A03 Software Supply Chain Failures | Dependency/SBOM and CI/CD review, not normal web scans | Runtime web traffic alone cannot verify build provenance |
| A04 Cryptographic Failures | Browser/TLS inspection, configuration review | Proper key management and data-at-rest protection |
| A05 Injection | sqlmap; carefully controlled proxy/DAST tests | All context-specific injection paths or exploit impact |
| A06 Insecure Design | Manual workflow and abuse-case testing | Design intent without requirements and architecture evidence |
| A07 Authentication Failures | Designated-account login/session tests | Account lifecycle and identity-provider controls in full |
| A08 Software/Data Integrity Failures | Update/upload flow review; configuration inspection | Integrity guarantees without source, pipeline, and signing evidence |
| A09 Logging & Alerting Failures | Compare known Kali actions with collected telemetry | Retention, SOC response, and alert quality without operational review |
| A10 Mishandling Exceptional Conditions | Bounded malformed-input and load tests | Safe behavior under all failure modes or large-scale load |

Reference: [OWASP Top 10:2025](https://owasp.org/Top10/)

## 12. A Reproducible Collection Plan

Collect one traffic class at a time. The following order creates a clean and useful dataset.

1. **Record configuration** — note VM versions, target IP, tool versions, Suricata ruleset version, and Nginx configuration.
2. **Start collection** — complete [Start.md](Start.md), including packet capture.
3. **Baseline browsing** — browse pages and perform simple product searches at human speed.
4. **Low-rate reconnaissance** — run one bounded Nmap or WhatWeb task.
5. **Content discovery** — run one small wordlist with a low rate limit.
6. **Controlled validation** — run one separately labelled SQLi, XSS, authentication, or access-control scenario.
7. **Bounded load comparison** — only when needed, use a modest `ab` or `wrk` profile.
8. **Cool down** — wait briefly and make a final health-check request.
9. **Export evidence** — use [End.md](End.md) to stop capture and archive logs.
10. **Create a manifest** — record what occurred during each time window.

### Recommended collection manifest

Save this information beside every exported archive:

```text
run_id: content-discovery-001
date_time_start: 2026-08-26T14:10:00+07:00
date_time_end: 2026-08-26T14:15:00+07:00
source: 192.168.100.20
target: 192.168.100.10
target_application: OWASP Juice Shop via Nginx
tool: ffuf
tool_version: <record output of ffuf -V>
command_or_profile: common wordlist, rate 10 requests/sec
expected_label: content-discovery-low-rate
authorization_reference: local training lab
notes: packet capture started before the run; no config changes during run
```

## 13. Correlation and Analysis Workflow

When investigating an event, correlate in this order:

1. Start with the run manifest: identify the expected activity and time window.
2. Search `nginx_access.log` for the Kali source IP, path, or unique test marker.
3. Search `suricata_eve.json` for the same source/destination pair and nearby timestamps.
4. Open the matching PCAP window in Wireshark or inspect it with TShark.
5. Review `juiceshop_app.log` for related application behavior.
6. Classify the event as expected, unexpected, false positive, false negative, or inconclusive.

Example commands for local analysis on Kali after exporting the archive:

```bash
jq 'select(.event_type == "alert")' suricata_eve.json
tshark -r attack_traffic.pcap -Y 'http.request' \
  -T fields -e frame.time -e ip.src -e http.request.method -e http.request.full_uri
```

If no Suricata alert appears, do not assume the request was harmless. Check whether the relevant rule was enabled, whether the traffic used the monitored interface, and whether the event type is configured for EVE output.

## 14. Dataset Quality and Integrity Checklist

- [ ] Packet capture began before Kali generated traffic.
- [ ] Each run has one primary traffic class and a recorded time window.
- [ ] Tool versions, key options, and wordlist names are recorded.
- [ ] Normal traffic and authorized test traffic are labelled separately.
- [ ] Kali and Ubuntu clocks use consistent time/timezone settings.
- [ ] Sensitive values such as passwords, session cookies, and tokens are removed or protected before sharing.
- [ ] The exported archive is checked before cleanup.
- [ ] A checksum is recorded when the dataset is transferred or shared.

Optional integrity check after creating the dataset archive on Ubuntu:

```bash
sha256sum ~/dataset_*.tar.gz
```

## 15. Common Mistakes to Avoid

| Mistake | Why it harms the dataset | Better approach |
| --- | --- | --- |
| Running several scanners at once | Makes attribution and labels ambiguous | Run one tool/scenario per window |
| Using an unrestricted wordlist or maximum speed | Overwhelms the target and distorts traffic | Limit wordlist, rate, duration, and concurrency |
| Treating scanner output as proof | Produces false positives and overconfident conclusions | Verify with logs, PCAP, and intended lab behavior |
| Capturing only PCAP | Loses HTTP/application context | Archive PCAP, Nginx, Suricata, and app logs together |
| Changing Suricata rules mid-run | Prevents fair comparisons between runs | Freeze configuration or document every change |
| Publishing raw logs without review | Can expose credentials, tokens, or personal data | Redact, protect, and control access to the archive |
| Treating Nginx as a WAF by default | Misstates what is being tested | Distinguish reverse-proxy logging from configured WAF controls |

## 16. References

1. [OWASP Top 10:2025](https://owasp.org/Top10/)
2. [OWASP Web Security Testing Guide v4.2](https://owasp.org/www-project-web-security-testing-guide/v42/)
3. [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/)
4. [Nmap Reference Guide](https://nmap.org/book/man.html)
5. [Nmap Service and Version Detection](https://nmap.org/book/vscan.html)
6. [PortSwigger — Burp Repeater](https://portswigger.net/burp/documentation/desktop/tools/repeater)
7. [PortSwigger — Insecure Direct Object References](https://portswigger.net/web-security/access-control/idor)
8. [OWASP ZAP Documentation](https://www.zaproxy.org/docs/)
9. [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
10. [OWASP Cross Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
11. [Kali Linux Package Repository Guidance](https://www.kali.org/docs/general-use/kali-apt-sources/)
12. [Kali Linux Metapackages](https://www.kali.org/docs/general-use/metapackages/)
13. [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
