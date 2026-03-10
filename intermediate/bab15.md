# BAB 15 — Metodologi Recon & Advanced Cheatsheet

> **NMAP Intermediate Guide · 2026 Edition**

---

## Metodologi Recon Terstruktur

### Fase 0: Persiapan

```bash
# Buat workspace
mkdir -p ~/pentest/target/{nmap,screenshots,loot,notes}
cd ~/pentest/target

# Tentukan scope
echo "192.168.1.0/24" > scope.txt
echo "192.168.1.5" >> exclude.txt  # Server yang tidak boleh di-scan

# Setup logging
exec > >(tee -a engagement.log) 2>&1
echo "=== Mulai: $(date) ==="
```

### Fase 1: Passive Recon (tanpa mengirim paket ke target)

```bash
# List scan: hanya DNS lookup, tidak ada probe
nmap -sL 192.168.1.0/24 -oN 00_list_scan.txt

# Baca seluruh hostname
grep "Nmap scan report" 00_list_scan.txt
```

### Fase 2: Host Discovery (light touch)

```bash
sudo nmap -sn -T3 \
  --excludefile exclude.txt \
  192.168.1.0/24 \
  -oA 01_host_discovery

grep "up" 01_host_discovery.nmap | awk '{print $NF}' | \
  tr -d '()' > live_hosts.txt

echo "Host aktif: $(wc -l < live_hosts.txt)"
```

### Fase 3: Port Enumeration

```bash
# Quick scan (top 1000 port)
sudo nmap -sS -T4 --open \
  -iL live_hosts.txt \
  -oA 02_port_scan_quick

# Full scan jika diperlukan
sudo nmap -p- -sS -T4 --open --min-rate 2000 \
  -iL live_hosts.txt \
  -oA 02_port_scan_full
```

### Fase 4: Service & Version Detection

```bash
sudo nmap -sS -sV -sC -T4 --open \
  -iL live_hosts.txt \
  -oA 03_service_scan
```

### Fase 5: Targeted Deep Scan

```bash
# Berdasarkan temuan fase 4, scan lebih dalam pada layanan menarik
# Contoh: web servers
grep "80/open\|443/open\|8080/open" 02_port_scan_quick.gnmap | \
  awk '{print $2}' > web_hosts.txt

sudo nmap --script "http-enum,http-title,http-headers,http-security-headers" \
  -sV -p 80,443,8080,8443 \
  -iL web_hosts.txt \
  -oA 04_web_scan
```

### Fase 6: Vulnerability Assessment

```bash
# Gunakan script vuln dengan hati-hati (lebih intrusif)
sudo nmap --script vuln --open -T4 \
  -iL live_hosts.txt \
  -oA 05_vuln_scan
```

---

## Keputusan Scan Berdasarkan Konteks

```
Apakah kamu punya izin tertulis?
├── TIDAK → BERHENTI. Jangan scan.
└── YA
    ├── Jaringan produksi?
    │   ├── YA → Gunakan timing T2-T3, hindari --script vuln/brute
    │   └── TIDAK (test/dev/lab) → T4 aman
    │
    ├── Apakah IDS/IPS aktif?
    │   ├── YA (stealth mode) → T1, -f, -D RND:5, --randomize-hosts
    │   └── TIDAK → T4 aman
    │
    └── Berapa lama time window?
        ├── < 1 jam → Top 1000 port, no version scan
        ├── 1-4 jam → Top 1000 + version detection
        └── > 4 jam → Full port scan (p-) + version + scripts
```

---

## Advanced Cheatsheet

### Scan Types

```bash
sudo nmap -sS target       # SYN Stealth (default terbaik)
sudo nmap -sT target       # TCP Connect (tanpa root)
sudo nmap -sU target       # UDP scan
sudo nmap -sN target       # Null scan (evade firewall)
sudo nmap -sF target       # FIN scan
sudo nmap -sX target       # Xmas scan
sudo nmap -sA target       # ACK scan (map firewall)
sudo nmap -sI zombie target  # Idle/Zombie scan
sudo nmap -sO target       # IP Protocol scan
sudo nmap -sY target       # SCTP INIT scan
```

### Discovery

```bash
nmap -sn target/24         # Ping scan (host discovery)
nmap -sL target/24         # List scan (DNS only, no probe)
nmap -Pn target            # Skip discovery (assume up)
nmap -PR target/24         # ARP scan (local network)
nmap -PE target            # ICMP Echo ping
nmap -PS80,443 target      # TCP SYN ping
sudo nmap -6 -sn target    # IPv6 discovery
```

### Port Specification

```bash
nmap -p 80 target          # Single port
nmap -p 22,80,443 target   # Multiple ports
nmap -p 1-1000 target      # Port range
nmap -p- target            # All 65535 ports
nmap --top-ports 100 target # Top 100 ports
nmap --open target         # Only show open ports
nmap -p U:161,T:22 target  # UDP + TCP mix
```

### Detection & Enumeration

```bash
nmap -sV target            # Version detection
nmap -sV --version-all target  # Maximum version probes
nmap -O target             # OS detection
nmap -A target             # All: OS+Version+Scripts+Traceroute
nmap -sC target            # Default scripts
nmap --script vuln target  # Vulnerability scripts
nmap --script "http-*" -p 80 target    # All HTTP scripts
nmap --script "smb-*" -p 445 target    # All SMB scripts
```

### Timing & Performance

```bash
nmap -T0 target            # Paranoid (5 menit/probe)
nmap -T1 target            # Sneaky (15 detik/probe)
nmap -T2 target            # Polite (0.4 detik/probe)
nmap -T3 target            # Normal (default)
nmap -T4 target            # Aggressive (fast)
nmap -T5 target            # Insane (very fast, may miss)
nmap --min-rate 1000 target  # Min 1000 pkt/sec
nmap --max-retries 1 target  # Sedikit retry = cepat
nmap --host-timeout 30s target  # Abandon slow hosts
```

### Evasion

```bash
nmap -f target             # Fragment packets (8 byte)
nmap --mtu 24 target       # Custom fragment size
nmap -D RND:10 target      # 10 random decoys
nmap -D 1.2.3.4,ME target  # Specific decoy
nmap -g 53 target          # Source port 53
nmap --spoof-mac 0 target  # Random MAC
nmap --data-length 25 target  # Padding
nmap --randomize-hosts target/24  # Random order
```

### Output

```bash
nmap -oN file.txt target   # Normal text
nmap -oX file.xml target   # XML
nmap -oG file.gnmap target # Grepable
nmap -oA basename target   # All formats
nmap -v target             # Verbose
nmap -vv target            # Very verbose
nmap -d target             # Debug
nmap --stats-every 10s target  # Progress updates
```

### NSE Script Control

```bash
nmap --script default target        # Default scripts
nmap --script "not brute" target    # Exclude brute
nmap --script "vuln or exploit" target
nmap --script http-title --script-args http.useragent="Mozilla" target
nmap --script-help http-enum        # Baca docs script
nmap --script-updatedb              # Update database
```

### IPv6

```bash
nmap -6 ::1                # Scan IPv6
nmap -6 -sS fe80::1%eth0   # Link-local
nmap -6 -A 2001:db8::1     # Aggressive IPv6
```

---

## Daftar Port Penting untuk Diingat

| Port       | Service           | Tips                               |
|------------|-------------------|------------------------------------|
| 21         | FTP               | Check anonymous login              |
| 22         | SSH               | Version exploit, weak creds        |
| 23         | Telnet            | Plaintext, sering di IoT           |
| 25,587     | SMTP              | Open relay check                   |
| 53         | DNS               | Zone transfer, brute subdomain     |
| 80,443     | HTTP/HTTPS        | Web vuln, expired cert             |
| 88         | Kerberos          | Indikasi Active Directory          |
| 110,995    | POP3              | Email enumeration                  |
| 139,445    | SMB               | EternalBlue, enum shares/users     |
| 143,993    | IMAP              | Email enumeration                  |
| 161        | SNMP/UDP          | Community string "public"          |
| 389,636    | LDAP              | AD enumeration, null bind          |
| 1433       | MSSQL             | Default creds, xp_cmdshell         |
| 3306       | MySQL             | Empty password check               |
| 3389       | RDP               | BlueKeep, brute force              |
| 5432       | PostgreSQL        | Default creds                      |
| 5900       | VNC               | No auth, weak password             |
| 6379       | Redis             | Unauthenticated access             |
| 8080,8443  | HTTP alt          | Dev server, admin panels           |
| 27017      | MongoDB           | Unauthenticated access             |

---

## Penutup Modul Intermediate

Kamu telah mempelajari:
- Teknik scan lanjutan (Idle scan, SCTP, IP Protocol scan)
- DNS enumeration dan network mapping mendalam
- Banner grabbing dan deep fingerprinting
- NSE scripting lanjutan dan penulisan custom script
- Integrasi dengan Metasploit, Masscan, dan tools lain
- IPv6 scanning
- Optimasi untuk jaringan besar
- Parsing output dengan Python
- Evasion techniques
- Automasi dengan Bash dan Cron
- Metodologi recon terstruktur

**Langkah selanjutnya:**
- Praktik di platform seperti **HackTheBox**, **TryHackMe**, atau **VulnHub**
- Pelajari penulisan NSE script lebih dalam (baca source Lua dari `/usr/share/nmap/scripts/`)
- Pelajari **Wireshark** untuk memahami setiap paket yang Nmap kirim
- Pelajari **Metasploit Framework** secara mendalam
- Pertimbangkan sertifikasi: **CEH**, **OSCP**, **eJPT**

---

> **PERINGATAN TERAKHIR:** Semua teknik dalam modul ini hanya untuk digunakan pada sistem yang kamu miliki atau dengan izin tertulis yang eksplisit. Penggunaan ilegal dapat mengakibatkan tuntutan pidana.

---

*← [BAB 14: CTF & Pentest Scenarios](bab14.md)*

*Written for educational purposes · 2026 Edition*
