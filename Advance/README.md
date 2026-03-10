# NMAP Advanced Module — 2026 Edition

> Panduan Nmap tingkat **Advanced** — untuk security researcher, penetration tester, dan red teamer berpengalaman.

---

## Prasyarat

Sebelum memulai modul ini, pastikan kamu sudah menguasai:
- Seluruh materi modul **Beginner** (dasar Nmap, sintaks, output, NSE dasar)
- Seluruh materi modul **Intermediate** (TCP/IP lanjut, scan teknik, Metasploit integration, evasion dasar)
- Linux command line profisien
- Python 3.x dan Lua dasar
- Konsep jaringan: TCP/IP stack, DNS, TLS, HTTP

---

## Struktur Modul

| Bab | Topik | Level |
|-----|-------|-------|
| [BAB 01](bab01.md) | Nmap Architecture & Internals | ⬛⬛⬛⬛⬜ |
| [BAB 02](bab02.md) | Advanced OS & Service Fingerprinting | ⬛⬛⬛⬛⬜ |
| [BAB 03](bab03.md) | Advanced NSE Lua Programming | ⬛⬛⬛⬛⬜ |
| [BAB 04](bab04.md) | Protocol-Level Packet Crafting | ⬛⬛⬛⬛⬛ |
| [BAB 05](bab05.md) | Attacking & Analyzing Encrypted Services | ⬛⬛⬛⬛⬜ |
| [BAB 06](bab06.md) | Advanced Active Directory Reconnaissance | ⬛⬛⬛⬛⬛ |
| [BAB 07](bab07.md) | Cloud & Container Environment Scanning | ⬛⬛⬛⬛⬜ |
| [BAB 08](bab08.md) | ICS/SCADA & IoT Scanning | ⬛⬛⬛⬛⬛ |
| [BAB 09](bab09.md) | Evading Modern EDR, NGFW & IDS | ⬛⬛⬛⬛⬜ |
| [BAB 10](bab10.md) | Building a Scan Automation Pipeline | ⬛⬛⬛⬛⬜ |
| [BAB 11](bab11.md) | Nmap dalam Red Team Operations | ⬛⬛⬛⬛⬛ |
| [BAB 12](bab12.md) | Performance Tuning & Internals | ⬛⬛⬛⬜⬜ |
| [BAB 13](bab13.md) | Threat Hunting dengan Nmap (Blue Team) | ⬛⬛⬛⬛⬜ |
| [BAB 14](bab14.md) | Building Custom Vulnerability Scanner | ⬛⬛⬛⬛⬛ |
| [BAB 15](bab15.md) | Advanced Methodology & Master Cheatsheet | ⬛⬛⬛⬛⬜ |

> ⬛ = tingkat kesulitan / kedalaman materi

---

## Apa yang Akan Kamu Pelajari

### Nmap Internals (BAB 01-03)
- Source code architecture C++ Nmap
- UltraScan algorithm dan congestion windows
- Semua 18 probe OS fingerprinting dan cara kerjanya
- Cara menulis custom NSE scripts dengan coroutine, multi-threading, dan `brute.Driver`

### Protocol-Level Mastery (BAB 04-05)
- Craft raw TCP/UDP/ICMP packets via NSE `packet` library
- Scapy integration untuk paket malformed
- TLS/SSL cipher audit, STARTTLS enumeration, JA3 fingerprinting
- CVE scanning: ssl-heartbleed, ssl-poodle, ssl-ccs-injection

### Enterprise Environment (BAB 06-08)
- Active Directory full recon: DNS, LDAP, Kerberos, SMB, RPC, WinRM
- ASREPRoast dan Kerberoast detection setup
- Cloud metadata endpoints, Docker API, Kubernetes API, etcd enumeration
- ICS/SCADA industrial protocols: Modbus, Siemens S7, EtherNet/IP, BACnet, DNP3

### Advanced Techniques (BAB 09-12)
- Bypass NGFW application-layer inspection, IDS signature evasion, EDR detection
- Full Python scan automation pipeline dengan SQLite, Slack alerting, CI/CD
- Red team OPSEC: pivot via SOCKS5, static binary deployment, living off the land
- libpcap buffer tuning, Masscan+Nmap pipeline, scan performance benchmarking

### Blue Team & Engineering (BAB 13-15)
- Network baselining dengan ndiff, automated change detection
- SIEM integration (Elasticsearch), compliance scanning
- Custom vulnerability scanner: NVD API + CVE matcher + HTML reporter
- Master cheatsheet semua flag, decision tree methodology

---

## Setup Tools yang Dibutuhkan

```bash
# Wajib
sudo apt-get install -y nmap masscan python3 python3-pip

# Python dependencies
pip3 install requests scapy python-libnmap

# Optional tapi sangat berguna
sudo apt-get install -y \
    wireshark tshark \     # Packet analysis
    ldap-utils \          # ldapsearch, ldapmodify
    bloodhound \          # AD recon visualization
    seclists \            # Wordlists untuk NSE brute
    zaproxy               # Web proxy untuk TLS analysis

# Nmap NSE scripts tambahan
sudo nmap --script-updatedb

# Verifikasi setup
nmap --version
python3 -c "import scapy; print('scapy ok')"
masscan --version
```

---

## Etika & Legal

> **Hanya gunakan ilmu ini pada sistem yang kamu miliki atau telah mendapat izin tertulis eksplisit.**

Beberapa bab (terutama BAB 06, 08, 11) membahas teknik yang sangat powerful. Semua contoh command **harus dijalankan di lab environment** atau dalam scope engagement yang sah.

Referensi framework legal:
- **Bug Bounty**: Ikuti scope program (HackerOne, Bugcrowd)
- **Penetration Test**: Wajib ada Statement of Work (SOW) dan Rules of Engagement
- **Red Team**: Wajib ada NDA + ROE + Get-Out-of-Jail letter
- **ICS/SCADA**: Koordinasi mandatory dengan tim operasional

---

## Tiga Seri Modul

| Modul | Lokasi | Target Reader |
|-------|--------|---------------|
| Beginner | [`../beginner/`](../beginner/README.md) | Baru mulai, tidak ada background IT |
| Intermediate | [`../intermediate/`](../intermediate/README.md) | IT/networking dasar, pernah pakai Nmap |
| **Advanced** | **`./`** (kamu di sini) | Pentest experience, scripting bisa |

---

*Mulai dari [BAB 01: Nmap Architecture & Internals](bab01.md)*
