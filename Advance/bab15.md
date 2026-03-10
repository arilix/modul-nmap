# BAB 15 — Advanced Methodology & Master Cheatsheet

> **NMAP Advanced Guide · 2026 Edition**

---

## Full Advanced Recon Methodology

```
┌─────────────────────────────────────────────────────────────┐
│              ADVANCED NMAP RECON FRAMEWORK                   │
│                                                              │
│  Phase 0: OPSEC Setup                                        │
│    └── VPN/proxy chain, clean identity, log hygiene         │
│                                                              │
│  Phase 1: Passive Intelligence                               │
│    └── WHOIS, ASN, CT logs, Shodan, DNS records             │
│                                                              │
│  Phase 2: Host Discovery                                     │
│    └── Ping sweep, ARP scan, DNS PT query                   │
│                                                              │
│  Phase 3: Port Reconnaissance                                │
│    └── Masscan fast sweep → Nmap accurate verify            │
│                                                              │
│  Phase 4: Service Deep Fingerprinting                        │
│    └── Version detection, banner grab, NSE scripts          │
│                                                              │
│  Phase 5: Vulnerability Correlation                          │
│    └── CVE lookup, CVSS scoring, exploit search             │
│                                                              │
│  Phase 6: Reporting & Remediation                            │
│    └── HTML report, JIRA tickets, DefectDojo import         │
└─────────────────────────────────────────────────────────────┘
```

---

## Phase 0: OPSEC Checklist

```bash
# Sebelum scan:
□ Pastikan VPN aktif: curl -s ifconfig.me  
□ Verify DNS tidak leak: curl -s https://dnsleaktest.com/api/v1/leak  
□ Setup log dir terpisah: mkdir -p ~/ops/$(date +%Y%m%d)  
□ Catat IP source kamu saat ini untuk laporan  
□ Dapatkan scope tertulis (IP range yang diizinkan)  
□ Koordinasi dengan defender jika diperlukan  
□ Backup konfigurasi target jika ada akses  

# Cek current public IP
curl -s https://ipinfo.io/json | python3 -m json.tool | grep '"ip"\|"org"'
```

---

## Phase 1: Passive Intelligence

```bash
# --- WHOIS / IP Range ---
whois target.com | grep -i "netrange\|cidr\|netname\|org"
whois AS12345 | grep "route"

# --- Certificate Transparency ---
curl -s "https://crt.sh/?q=%.target.com&output=json" |
python3 -c "import json,sys; [print(x['name_value']) for x in json.load(sys.stdin)]" |
sort -u | tee ct_domains.txt

# --- DNS SRV / Zone Transfer ---
dig +short @ns1.target.com target.com AXFR 2>/dev/null || echo "Zone transfer blocked"
dig _ldap._tcp.target.com SRV 2>/dev/null

# --- Shodan CLI ---
shodan search "org:\"Target Corp\"" --fields ip_str,port,product | head -20

# --- ASN / BGP ---
curl -s "https://ipinfo.io/AS12345/json" | python3 -m json.tool
```

---

## Phase 2: Host Discovery

```bash
# --- ARP scan (Layer 2, lokal network) ---
sudo nmap -PR -sn 192.168.1.0/24

# --- ICMP Echo + TCP SYN hostdiscovery ---
sudo nmap -PE -PS22,80,443 -sn 10.0.0.0/8 -T4 --min-rate 5000 -oG hosts.gnmap
grep "Up" hosts.gnmap | awk '{print $2}' > alive_hosts.txt
echo "[+] Hosts found: $(wc -l < alive_hosts.txt)"

# --- DNS-based discovery ---
nmap --script dns-brute --script-args dns-brute.domain=target.com -p 53 ns1.target.com

# --- ARP broadcast (local discovery) ---
sudo arping -c 3 192.168.1.1 2>/dev/null
sudo nmap --script broadcast-arp-ping 192.168.1.0/24
```

---

## Phase 3: Port Reconnaissance

```bash
# --- Step 3a: Masscan fast sweep (gunakan masscan jika ada) ---
sudo masscan $(cat alive_hosts.txt | tr '\n' ' ') \
     -p 0-65535 --rate 50000 --output-format list \
     --output-file masscan_raw.txt 2>/dev/null

# Parse masscan → nmap format
awk '/^open tcp/{printf "%s:%s\n", $4, $3}' masscan_raw.txt | sort -u > open_services.txt

# --- Step 3b: Nmap verify + topports jika masscan tidak tersedia ---
sudo nmap -sS -T4 --min-rate 2000 \
     -p "$(sort -un masscan_raw.txt | awk -F: '{print $2}' | tr '\n' ',' | sed 's/,$//')" \
     -iL alive_hosts.txt --open -oX verified_ports.xml

# --- Quick top ports (tanpa masscan) ---
sudo nmap -sS -T4 --top-ports 1000 -iL alive_hosts.txt --open -oX top1000.xml
```

---

## Phase 4: Service Deep Fingerprinting

```bash
# --- Version detection comprehensive ---
sudo nmap -sS -sV --version-all \
     -sC \
     --script "banner,version,default" \
     -p $(grep "open" verified_ports.xml | grep -oP 'portid="\K\d+' | sort -un | tr '\n' ',' | sed 's/,$//') \
     -iL alive_hosts.txt \
     -oA fingerprint_results

# --- OS detection ---
sudo nmap -O --osscan-guess \
     -iL alive_hosts.txt \
     -oX os_results.xml

# --- SSL/TLS audit ---
nmap --script ssl-enum-ciphers,ssl-cert,ssl-poodle,ssl-heartbleed \
     -p 443,8443,465,993,995,636,3269 \
     -iL alive_hosts.txt -oN ssl_audit.txt
```

---

## Phase 5: Vulnerability Correlation

```bash
# --- NSE vuln scripts ---
sudo nmap --script "vuln" -p $(cat open_ports.txt) \
     -iL alive_hosts.txt -oN vuln_results.txt --script-timeout 30s

# --- Searchsploit correlation ---
while read product version; do
    echo "=== $product $version ==="
    searchsploit "$product $version" --disable-colour 2>/dev/null | grep -v "^--\|^$" | head -5
done < <(grep "product\|version" fingerprint_results.xml |
         grep -oP '(?<=product=")[^"]+|(?<=version=")[^"]+' | paste - -)

# --- Custom vuln scanner (bab14) ---
python3 vuln_scanner.py "$(cat alive_hosts.txt | head -1)" > vuln_output.json
```

---

## Master Cheatsheet: Semua Flag & Option

### Scan Types

| Flag | Deskripsi | Privilege |
|------|-----------|-----------|
| `-sS` | TCP SYN (half-open) | root |
| `-sT` | TCP Connect (full) | any |
| `-sU` | UDP scan | root |
| `-sA` | TCP ACK (firewall mapping) | root |
| `-sW` | TCP Window scan | root |
| `-sM` | TCP Maimon scan | root |
| `-sN/-sF/-sX` | Null/FIN/Xmas scan | root |
| `-sY/-sZ` | SCTP INIT/Cookie-Echo | root |
| `-sO` | IP Protocol scan | root |
| `-sI zombie` | Idle/Zombie scan | root |
| `-b ftphost` | FTP bounce scan | any |

### Host Discovery

| Flag | Deskripsi |
|------|-----------|
| `-sn` | Ping scan only (no port scan) |
| `-Pn` | Skip host discovery |
| `-PS[ports]` | TCP SYN ping |
| `-PA[ports]` | TCP ACK ping |
| `-PU[ports]` | UDP ping |
| `-PE/-PP/-PM` | ICMP Echo/Timestamp/Mask |
| `-PR` | ARP ping |
| `--disable-arp-ping` | Gunakan IP scan bukan ARP |
| `-n` | Tidak DNS resolve |
| `-R` | Selalu DNS resolve |

### Port Specification

| Flag | Deskripsi |
|------|-----------|
| `-p 22,80,443` | Port tertentu |
| `-p 1-1024` | Range |
| `-p -` | Semua 65535 port |
| `--top-ports N` | N port terpopuler |
| `-F` | Fast: 100 port teratas |
| `--port-ratio 0.1` | Port dengan usage > 10% |
| `-r` | Sequential port order |
| `--exclude-ports` | Kecualikan port tertentu |

### Service & OS Detection

| Flag | Deskripsi |
|------|-----------|
| `-sV` | Version detection |
| `--version-intensity 0-9` | Intensitas (default 7) |
| `--version-light` | Intensity 2 |
| `--version-all` | Intensity 9 |
| `--version-trace` | Debug version detection |
| `-O` | OS detection |
| `--osscan-limit` | Hanya host mungkin |
| `--osscan-guess` | Aggressive OS guess |
| `-A` | Aggressive: OS+version+scripts+traceroute |

### NSE Scripting

| Flag | Deskripsi |
|------|-----------|
| `-sC` | Default scripts |
| `--script=<name/cat/dir/expr>` | Script tertentu |
| `--script-args key=val` | Script arguments |
| `--script-args-file file` | Args dari file |
| `--script-trace` | Debug script traffic |
| `--script-updatedb` | Update script database |
| `-d[1-9]` | Debug level |
| `--script-timeout Xs` | Max waktu per script |

### Timing & Performance

| Flag | Deskripsi |
|------|-----------|
| `-T0` to `-T5` | Template timing |
| `--min-rate N` | Minimum N packet/detik |
| `--max-rate N` | Maximum N packet/detik |
| `--min-parallelism N` | Minimum concurrent probes |
| `--max-parallelism N` | Maximum concurrent probes |
| `--scan-delay Xms/s` | Delay antar probes |
| `--max-scan-delay` | Max scan delay |
| `--host-timeout Xm` | Abandon host setelah X menit |
| `--min-rtt-timeout` | Minimum RTT wait |
| `--max-rtt-timeout` | Maximum RTT wait |
| `--initial-rtt-timeout` | Initial probe wait |
| `--max-retries N` | Port probe retransmissions |

### Evasion & Firewall Bypass

| Flag | Deskripsi |
|------|-----------|
| `-f` | Fragment packets (8 byte) |
| `-ff` | Double fragment (16 byte) |
| `--mtu N` | Set MTU (multiple of 8) |
| `-D decoy1,ME,decoy2` | Decoy IPs |
| `-D RND:N` | N random decoys |
| `-S <IP>` | Spoof source IP |
| `-e <iface>` | Use specific interface |
| `-g / --source-port N` | Spoof source port |
| `--data-length N` | Append N random bytes |
| `--data-string "str"` | Append string payload |
| `--data 0xhex` | Append hex data |
| `--ip-options opts` | Set IP options |
| `--ttl N` | Set TTL value |
| `--spoof-mac MAC/OUI/0` | Spoof MAC address |
| `--badsum` | Send bad checksum |
| `--randomize-hosts` | Randomize host scan order |
| `--proxies url1,url2` | Use HTTP/SOCKS proxy |

### Output

| Flag | Deskripsi |
|------|-----------|
| `-oN file` | Normal output |
| `-oX file` | XML output |
| `-oG file` | Grepable output |
| `-oA prefix` | All formats |
| `-oS file` | Script kiddie output |
| `-v / -vv` | Verbosity |
| `-d[1-9]` | Debug level |
| `--reason` | Why port is in state |
| `--open` | Only show open ports |
| `--packet-trace` | Show all sent/received packets |
| `--iflist` | List interfaces/routes |
| `--append-output` | Append to existing file |
| `--resume file` | Resume aborted scan |
| `--noninteractive` | Disable interactive input |
| `--stats-every Xs` | Print stats every X seconds |

---

## Decision Tree: Pilih Teknik yang Tepat

```
Ingin scan cepat untuk banyak host?
├── Ya → Masscan + Nmap phased approach (bab12)
└── Tidak → Lanjut

Lingkungan sangat terproteksi?
├── Ya → T1 + fragmentation + decoy + source port (bab09)
└── Tidak → T3-T4 standard

Target Windows AD environment?
├── Ya → SMB + LDAP + Kerberos + RPC (bab06)
└── Tidak → Lanjut

Target cloud/container?
├── Ya → Docker API + K8s + metadata endpoints (bab07)
└── Tidak → Lanjut

Target ICS/OT?
├── Ya → T1, max-rate 10, ICS-specific NSE (bab08)
└── Tidak → Standard approach

Butuh blue team monitoring?
├── Ya → ndiff + baseline + NetworkMonitor (bab13)
└── Tidak → Standard output

Perlu CVE correlation?
├── Ya → Custom vuln scanner (bab14)
└── Tidak → NSE vuln scripts: --script vuln
```

---

## NSE Script Categories Reference

| Kategori | Deskripsi | Contoh Script |
|----------|-----------|---------------|
| `auth` | Authentication bypass/brute | ssh-brute, ftp-anon |
| `broadcast` | Network discovery | broadcast-arp-ping, broadcast-upnp-info |
| `brute` | Credential brute force | http-brute, smtp-brute |
| `default` | -sC scripts | ssl-cert, ssh-hostkey |
| `discovery` | Network info gathering | dns-zone-transfer, ldap-rootdse |
| `dos` | Denial of Service (hati-hati!) | smb-vuln-ms10-054 |
| `exploit` | Actual exploitation | ms17-010-eternalblue |
| `external` | Third-party queries | whois-ip, http-google-malware |
| `fuzzer` | Fuzz protocol | dns-fuzz |
| `intrusive` | Potentially harmful | http-passwd, snmp-brute |
| `malware` | Malware detection | smtp-strangeport |
| `safe` | Won't harm target | banner, http-title |
| `version` | Version detection support | (internal) |
| `vuln` | Vulnerability scanning | ssl-heartbleed, smb-vuln-ms17-010 |

---

## Port Reference Guide

| Port | Service | Kerentanan Umum |
|------|---------|-----------------|
| 21 | FTP | Anonymous login, cleartext creds |
| 22 | SSH | Weak creds, CVE-2018-15473 (user enum) |
| 23 | Telnet | Cleartext, always disable |
| 25 | SMTP | Open relay, user enum |
| 53 | DNS | Zone transfer, cache poison |
| 80 | HTTP | Web vulns (OWASP Top 10) |
| 88 | Kerberos | ASREPRoast, Kerberoast |
| 110 | POP3 | Cleartext credentials |
| 139/445 | SMB | EternalBlue, relay attacks, null sessions |
| 143 | IMAP | Cleartext credentials |
| 389 | LDAP | Anonymous enum, injection |
| 443 | HTTPS | TLS misconfig, web vulns |
| 1433 | MSSQL | SA bruteforce, xp_cmdshell |
| 3306 | MySQL | Remote root, weak creds |
| 3389 | RDP | BlueKeep, credential brute |
| 5432 | PostgreSQL | Default postgres no-pass |
| 6379 | Redis | No auth, RCE via config write |
| 8080 | HTTP-Alt | Often dev/staging with weaker security |
| 27017 | MongoDB | No auth, command injection |

---

## Kesimpulan: Perjalanan Advanced

Setelah menyelesaikan modul ini, kamu telah menguasai:

1. **Internals** — Cara kerja Nmap dari source code hingga packet
2. **Fingerprinting** — OS/service detection mendalam dan custom
3. **NSE Mastery** — Scripting Lua tingkat lanjut dengan coroutines
4. **Packet Crafting** — Raw socket dan protocol manipulation
5. **Encrypted Services** — TLS/SSL analysis dan audit
6. **Active Directory** — Kerberos, LDAP, SMB enterprise recon
7. **Cloud/Container** — AWS, GCP, Azure, Docker, Kubernetes
8. **ICS/SCADA** — Protokol industri dan keamanan OT
9. **Evasion** — Bypass EDR, NGFW, dan IDS modern
10. **Automation** — Pipeline scanning berbasis CI/CD dan SIEM
11. **Red Team** — OPSEC, pivoting, living off the land
12. **Performance** — Tuning libpcap, masscan integration
13. **Threat Hunting** — Baseline, change detection, blue team
14. **Vuln Scanner** — Build custom CVE correlation engine
15. **Methodology** — Framework end-to-end dari rekon ke laporan

---

> **"Give a man a fish, you feed him for a day. Teach him Nmap, and he can map every fish in the ocean."**  
> — Security community proverb

---

*← [BAB 14: Custom Vulnerability Scanner](bab14.md) · [README](README.md)*
