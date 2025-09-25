# Week 5 — Network Forensics Lab (Endpoint → Middleware)

**CHFI v10 mapping:** Module 08 (Network Forensics)  
**Lecture alignment:** Capture → Reduce → Reconstruct → Correlate  
**Scope:** Host & application logs → packet capture → Zeek flows → DHCP/NAT attribution → firewall/proxy (middleware) → IDS validation (Suricata)  
**Note:** The Cloud Case (CloudTrail/VPC/Entra) is handled in a separate session — exclude cloud artifacts here.

---

## Learning Outcomes

By the end of 4 hours, students can:

- Perform network triage across **endpoint, packet, flow, and middleware** evidence.
- Handle **encrypted traffic** pragmatically (TLS/QUIC, DoH/DoT) using metadata and JA3/JA4.
- Attribute traffic behind NAT using **DHCP leases** and endpoint logon/process events.
- Explain **tool roles & blind spots** (Wireshark, Zeek, Suricata, firewall/proxy).
- Produce a **defensible report** with timeline (UTC), IOC matrix, hashes, and affidavit-style paragraph.

---

## Time Plan (120 min)

- Task 0 — Setup & Chain of Custody (10m)  
- Task 1 — Endpoint / Application Logs (20m)  
- Task 2 — Packet Triage with Wireshark (25m)  
- Task 3 — Zeek Flows & TLS Fingerprints (20m)  
- Task 4 — DHCP/NAT Attribution & Middleware Correlation (25m)  
- Task 5 — IDS Validation (Suricata) & Blind Spots (10m)  
- Task 6 — Package Report (10m)

Each task lists its **CHFI phase tag**.

---

## Dataset Layout (provided)

```ini
week1/
  pcaps/attack.pcap
  host-logs/
    windows/Security.evtx  Sysmon.evtx
    linux/syslog  auth.log
  app-logs/
    web/access.log  proxy/client.log
  infra-logs/
    dhcp/dhcp.log   firewall/fw.log   proxy/proxy.log
  ids/
    suricata/eve.json           # or empty if you will run Suricata locally
  working/
    evidence/                   # you will create outputs here
```

## Tools

Wireshark • Zeek • Suricata + `jq` • DB/Text viewers • PowerShell (`Get-WinEvent`) • `journalctl`/`awk` • hashing tools (`shasum`/`Get-FileHash`).

---

## Task 0 — Setup & Chain of Custody (10m) — *CHFI: Capture + Preservation*

### Steps

1. **Prepare directories**

   ```bash
   mkdir -p ~/week1/{pcaps,host-logs,app-logs,infra-logs,ids,working/evidence}
   cd ~/week1
   ```

2. **Install/verify tools**:
   - Wireshark: `wireshark --version`
   - Zeek: `zeek --version`
   - Suricata: `suricata --build-info`
   - jq: `jq --version`
   - DB Browser for SQLite if needed.
3. **Provision host/app logs**: Ensure `.evtx`, `syslog`, `auth.log`, `web/access.log`, `proxy/client.log` are available.
4. **Provision middleware logs**: DHCP (`dhcp.log`), firewall (`fw.log`), proxy (`proxy.log`).
5. **Hash inputs**:

   ```bash
   find pcaps host-logs app-logs infra-logs ids -type f -exec shasum -a 256 {} \; >> working/evidence/hashes.txt
   ```

6. **Document in lab_notes.md**: Analyst, timestamp, tools versions, hashes, handling rules.

**Checkpoint:** All tools verified, logs open, hashes recorded.

### Chain of Custody

1. **Create workspace**  

```bash
mkdir -p week1/working/evidence && cd week5
```

2. **Record timezone**: Use **UTC** in reporting; annotate local time (Asia/Ho_Chi_Minh, UTC+07) if needed.  

3. **Hash inputs** (append to `working/evidence/hashes.txt`):  

- Linux/macOS:  

  ```bash
  find pcaps host-logs app-logs infra-logs ids -type f -exec shasum -a 256 {} \; \
    >> working/evidence/hashes.txt
  ```

- Windows PowerShell:  

  ```powershell
  gci pcaps,host-logs,app-logs,infra-logs,ids -Recurse -File | \
    % { Get-FileHash $_.FullName -Algorithm SHA256 } | \
    Out-File working\evidence\hashes.txt -Append
  ```

4. **Start lab notes**: `working/evidence/lab_notes.md` — who/when/source paths/hashes, tool versions, read-only handling.

**Checkpoint:** Hash manifest & notes exist.

---

## Task 1 — Endpoint / Application Logs (20m) — *CHFI: Reduce*

### A) Windows (Security + Sysmon)

**Goal:** Identify logon sessions & suspicious processes near incident times.

```powershell
# Set your UTC window
$Start=(Get-Date "2025-09-18T01:00:00Z"); $End=(Get-Date "2025-09-18T02:00:00Z")
Get-WinEvent -FilterHashtable @{LogName='Security'; StartTime=$Start; EndTime=$End} |
  Export-Csv .\working\evidence\win_Security_window.csv -NoTypeInformation
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; StartTime=$Start; EndTime=$End} |
  Export-Csv .\working\evidence\win_Sysmon_window.csv -NoTypeInformation
```

**Hunt IDs:** Security 4624/4625/4672; Sysmon 1 (Process Create), 3 (Network), 11 (FileCreate), 22 (DNS).  
**Correlate:** Map **Sysmon EID 3** dstIP/dstPort to suspicious flows later; tie **EID 1** process/parent to user session (4624).

### B) Linux (auth/syslog)

```bash
journalctl --since "2025-09-18 01:00:00 UTC" --until "2025-09-18 02:00:00 UTC" \
  > working/evidence/linux_journal_window.txt
awk '/Sep 18 0(1|2):/ {print}' host-logs/linux/auth.log \
  > working/evidence/linux_auth_window.log
```

Look for SSH logins, sudo, process starts near suspicious network times.

### C) Application logs (client/web)

- **web/access.log**: large POSTs/long durations/odd URIs.  
- **proxy/client.log**: unusual CONNECTs, exotic User-Agents.

**Deliverable snippet:** `working/evidence/endpoint_findings.tsv` with `UTC,time,host,user,proc,event,notes`.

---

## Task 2 — Packet Triage with Wireshark (25m) — *CHFI: Reduce/Filter*

### Steps

1. Open `attack.pcap`.
2. Setup display filters:
   - HTTP: `http.request`, `http.request.method == "POST"`
   - DNS: `dns && dns.qry.name.len > 50`
   - TLS: `tls.handshake.type == 1`
   - QUIC: `quic || udp.port == 443`
   - DoT: `tcp.port == 853`
3. Use Statistics → Protocol Hierarchy, Conversations. Export CSV.
4. Export HTTP objects → `working/evidence/http_objects/`. Hash them.
5. Identify TLS SNI, JA3 values, flag anomalies.

**Deliverable:** Screenshots of filters/stats + exported object hashes.

### Wireshark filter

1) Open `pcaps/attack.pcap` in Wireshark. Create **display filter buttons**:

- General triage:  
  - `tcp`  
  - `udp`  
  - `tcp.analysis.flags`  
  - `frame.len > 1200`
- HTTP exfil:  
  - `http.request`  
  - `http.request.method == "POST"`  
  - `tcp.stream eq N` (after selecting a flow)
- DNS tunneling heuristics:  
  - `dns && dns.flags.response == 0`  
  - `dns.qry.name.len > 50`  
  - `dns.qry.name matches "(?i)[A-Za-z0-9]{20,}\."`
- Encrypted traffic:  
  - `tls.handshake.type == 1` (ClientHello)  
  - `tcp.port == 853` (DoT)  
  - `quic || (udp.port == 443)` (QUIC metadata)

2) **Statistics**: Endpoints, Conversations, Protocol Hierarchy — screenshot top talkers & anomalies.  
3) **Extract objects**: `File → Export Objects → HTTP` to `working/evidence/http_objects/`; hash exported files.  
4) **Notes**: Add suspected exfil hosts/URIs/DNS anomalies to `lab_notes.md` with UTC windows.

---

## Task 3 — Zeek Flows & TLS Fingerprints (20m) — *CHFI: Reconstruct*

1) **Run Zeek** on the pcap:

```bash
mkdir -p working/zeek-logs && cd working/zeek-logs
zeek -Cr ../../pcaps/attack.pcap local
```

2) **Readable views** (convert epoch to UTC with `-d`):

```bash
cat http.log | zeek-cut -d ts id.orig_h host uri method status_code > ../evidence/http_view.tsv
cat dns.log  | zeek-cut -d ts id.orig_h id.resp_h query qtype_name answers > ../evidence/dns_view.tsv
cat conn.log | zeek-cut -d ts id.orig_h id.resp_h id.resp_p proto service duration orig_bytes resp_bytes > ../evidence/conn_view.tsv
cat ssl.log  | zeek-cut -d ts id.orig_h id.resp_h id.resp_p server_name alpn ja3 ja3s > ../evidence/ssl_view.tsv
```

3) **TLS fingerprints** (find rare/odd JA3s):

```bash
cut -f8 ../evidence/ssl_view.tsv | sort | uniq -c | sort -nr > ../evidence/ja3_counts.txt   # adjust field index if needed
```

Investigate low-frequency JA3 values and unknown SNI; compare time windows with Wireshark flows.

4) **Flow fidelity note**: Record exporter timeouts (active/inactive) if provided; mention sampling if known; caution when estimating egress volume.

**Checkpoint:** Confirm indicators (hosts, URIs, queries, JA3/JA3S) and copy to IOC table.

---

## Task 4 — DHCP/NAT Attribution & Middleware Correlation (25m) — *CHFI: Reconstruct → Correlate*

### A) DHCP/NAT Attribution

1) **Goal:** Map suspicious internal IPs to **MAC/hostname/user** near the incident window.  
2) **Parse DHCP leases** (format varies; example regex-based grep):

```bash
# Example: lines containing "DHCPACK" with IP, MAC, hostname
grep -E "DHCPACK|Assigned" infra-logs/dhcp/dhcp.log | \
  awk '{print $1"T"$2, $0}' > working/evidence/dhcp_window.txt
```

3) **Join with Zeek**: For each suspicious `id.orig_h` (internal) from `conn_view.tsv`, find the nearest DHCP lease before the flow start. Record `MAC` and `hostname`.

### B) Firewall/Proxy Corroboration

1) **Firewall**: filter `fw.log` by internal IP and UTC window; confirm allow/deny and destination tuples.  
2) **Proxy/Gateway**: find matching `CONNECT`/HTTP(S) entries (hostnames, bytes, user agents).  
3) **Cross-verify** with Wireshark/Zeek times; screenshot a corroborating entry.

**Deliverable snippet:** `working/evidence/middleware_findings.tsv` with `UTC,srcIP,host(MAC),dst,port,device(log),decision,notes`.

---

## Task 5 — IDS Validation (Suricata) & Blind Spots (10m) — *CHFI: Correlate*

If you need to run Suricata yourself:

```bash
mkdir -p working/suri && suricata -r pcaps/attack.pcap -l working/suri -k none
```

Parse EVE JSON with `jq` (examples):

```bash
jq -c 'select(.event_type=="alert") | {ts:.timestamp,src:.src_ip,dst:.dest_ip,sig:.alert.signature}' \
  working/suri/eve.json > working/evidence/eve_alerts.jsonl
jq -c 'select(.event_type=="dns") | {ts:.timestamp,rrname:.dns.rrname,rcode:.dns.rcode}' \
  working/suri/eve.json > working/evidence/eve_dns.jsonl
```

**Reflection (2–3 bullets):**

- Which Suricata alerts **support** your Zeek/Wireshark findings?  
- What **alerts were missing** despite suspicious activity?  
- Identify one **blind spot** for each tool and state a mitigation.

---

## Task 6 — Package Your Report (10m)

### A) IOC Matrix (CSV)

Create `working/evidence/ioc_matrix.csv` with columns:

```
Source(Host/Zeek/DHCP/Firewall/Proxy/Suricata), Item, Value, Evidence Value (what it proves), Hash(if file)
```

*Example rows:*

```
Zeek/http, URI, /upload.php, Confirms exfil end-point, 
Zeek/ssl, JA3, 72a589da586844d7f0818ce684948eea, Rare client TLS fingerprint used during exfil, 
DHCP, Lease, 10.0.5.23→00:16:3e:aa:bb:cc (host lab-ws12), Attributed flows to host lab-ws12 at time T, 
Firewall, Allow, 10.0.5.23→198.51.100.10:443, Confirms egress path permitted, 
```

### B) Timeline (UTC) — include CHFI phase per row

`working/evidence/timeline.csv` columns:

```
UTC, Phase(Capture/Reduce/Reconstruct/Correlate), Source, Detail
```

### C) Affidavit-style paragraph (paste into `lab_notes.md`)
>
> On 2025‑09‑18 02:30 UTC I, <Name>, examined the dataset “week1.” Source files were hashed prior to analysis (SHA‑256 recorded in `working/evidence/hashes.txt`). I conducted packet triage in Wireshark, derived Zeek logs (http/dns/ssl/conn), inspected host/middleware logs (Windows/Linux, DHCP, firewall, proxy), and validated indicators with Suricata EVE JSON. All timestamps were normalized to UTC; observed clock skew is noted in the timeline. No writes were performed to originals; analysis outputs are under `working/evidence/`. To the best of my knowledge, this report is accurate and complete within the constraints documented.

### D) Submission Checklist

- [ ] Hash manifest & lab notes (CoC, tools, timezone)  
- [ ] Endpoint/application findings TSV  
- [ ] Wireshark screenshots (filters, conversations, object export)  
- [ ] Zeek views (http/dns/conn/ssl) + `ja3_counts.txt`  
- [ ] DHCP + firewall/proxy corroboration TSV  
- [ ] Suricata alerts JSONL + reflection bullets  
- [ ] IOC matrix CSV + timeline CSV (with CHFI phase)  
- [ ] Affidavit paragraph in `lab_notes.md`

---

## Quick Reference (copy/paste)

**Wireshark filters**

- HTTP: `http.request` • `http.request.method == "POST"` • `tcp.stream eq N` • `frame.len > 1200`  
- DNS tunneling: `dns && dns.flags.response == 0` • `dns.qry.name.len > 50` • regex for high entropy  
- TLS/QUIC/DoH/DoT: `tls.handshake.type == 1` • `quic || udp.port == 443` • `tcp.port == 853`

**Zeek commands**

```bash
zeek -Cr pcaps/attack.pcap local
cat ssl.log | zeek-cut -d ts id.orig_h id.resp_h id.resp_p server_name alpn ja3 ja3s > ssl_view.tsv
```

**Suricata + jq**

```bash
suricata -r pcaps/attack.pcap -l working/suri -k none
jq -c 'select(.event_type=="alert")' working/suri/eve.json | head
```

**Windows hashing**

```powershell
Get-FileHash .\path\to\file -Algorithm SHA256 | Out-File working\evidence\hashes.txt -Append
```

**Linux timezone tip**

```bash
date -u   # show current UTC; annotate offsets in your notes
```

---

## One-Minute Self-Check (before submission)

- Which **single artifact** best ties a human user to the suspicious flow?  
- What’s one **limitation** you faced with encrypted traffic, and how did you compensate?  
- If your flow-byte counts seemed off, how would **sampling/timeout** settings explain it?
