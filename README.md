<div align="center">

<img src="https://nmap.org/images/sitelogo.png" alt="Nmap Logo" width="200"/>

# NMAP Learning Module — 2026 Edition

![Nmap](https://img.shields.io/badge/Nmap-7.95-brightgreen?style=for-the-badge&logo=gnubash&logoColor=white)
![Level](https://img.shields.io/badge/Level-Beginner%20→%20Advanced-blue?style=for-the-badge)
![Language](https://img.shields.io/badge/Bahasa-Indonesia-red?style=for-the-badge)
![Bab](https://img.shields.io/badge/Total%20Bab-45-orange?style=for-the-badge)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)](LICENSE)
![Untuk](https://img.shields.io/badge/Untuk-Edukasi%20%26%20Lab-orange?style=for-the-badge)
[![Contributing](https://img.shields.io/badge/Contributing-Welcome-blueviolet?style=for-the-badge)](CONTRIBUTING.md)

> Panduan belajar Nmap lengkap dalam tiga tingkatan: **Beginner**, **Intermediate**, dan **Advanced**.  
> Semua materi ditulis dalam Bahasa Indonesia dengan istilah teknis bahasa Inggris dipertahankan.

</div>

---

## Pilih Level Kamu

| Level | Folder | Target Pembaca | Jumlah Bab |
|-------|--------|----------------|------------|
| 🟢 [Beginner](beginner/README.md) | `beginner/` | Pemula, tidak ada background IT | 15 Bab |
| 🟡 [Intermediate](intermediate/README.md) | `intermediate/` | Paham dasar jaringan, pernah pakai Nmap | 15 Bab |
| 🔴 [Advanced](Advance/README.md) | `Advance/` | Pengalaman pentest, bisa scripting | 15 Bab |

---

## Ringkasan Konten

### 🟢 Beginner (`beginner/`)

Cocok untuk yang baru pertama kali mengenal Nmap dan konsep scanning jaringan.

| Bab | Topik |
|-----|-------|
| BAB 01 | Apa itu Nmap? Sejarah dan kenapa belajar |
| BAB 02 | Instalasi di Linux, Windows, macOS |
| BAB 03 | Sintaks dasar, target specification, memahami output |
| BAB 04 | Host discovery (ping, ARP, -Pn) |
| BAB 05 | Teknik port scanning (SYN, Connect, UDP, Null/FIN/Xmas) |
| BAB 06 | Status port: open, closed, filtered, unfiltered |
| BAB 07 | Service & version detection (-sV) |
| BAB 08 | OS detection (-O, -A) |
| BAB 09 | NSE Scripting dasar (kategori dan script populer) |
| BAB 10 | Output dan reporting (format, menyimpan hasil) |
| BAB 11 | Timing & performance (template T0-T5) |
| BAB 12 | Evasion dasar (fragmentation, decoy, spoofing) |
| BAB 13 | Zenmap GUI (profil, fitur) |
| BAB 14 | Use case nyata (audit, vulnerability assessment) |
| BAB 15 | Cheat sheet & penutup legal |

---

### 🟡 Intermediate (`intermediate/`)

Untuk yang sudah paham dasar Nmap dan ingin mendalami teknik lebih lanjut.

| Bab | Topik |
|-----|-------|
| BAB 01 | TCP/IP deep dive, raw socket, BPF, TCP flags, probe DB |
| BAB 02 | Scan lanjut: Idle/Zombie (-sI), SCTP, IP Protocol, FTP Bounce |
| BAB 03 | Advanced target spec, DNS brute, zone transfer, traceroute |
| BAB 04 | Banner grabbing, fingerprinting HTTP/SSL/SMB/SNMP/DB/IoT |
| BAB 05 | NSE boolean expressions, --script-args, debugging |
| BAB 06 | Menulis NSE script Lua (struktur, library shortport/http/stdnse) |
| BAB 07 | Nmap + Metasploit integration (db_import, db_nmap) |
| BAB 08 | IPv6 scanning (ICMPv6, NDP, link-local, dual-stack) |
| BAB 09 | Scanning jaringan besar (phased strategy, masscan pipeline) |
| BAB 10 | Parsing XML dengan Python (libnmap, CSV/HTML report, ndiff) |
| BAB 11 | Advanced evasion (timing, fragmentation, decoy, MAC spoof) |
| BAB 12 | Automasi dengan Bash & Cron (recon workflow, Slack alerting) |
| BAB 13 | Nmap + tools lain (Masscan, EyeWitness, Nikto, Nuclei) |
| BAB 14 | CTF & pentest scenarios (web enum, AD pentest, pivot) |
| BAB 15 | Metodologi rekon & advanced cheatsheet |

---

### 🔴 Advanced (`Advance/`)

Untuk security researcher, penetration tester, dan red teamer berpengalaman.

| Bab | Topik |
|-----|-------|
| BAB 01 | Nmap architecture & internals (source code, UltraScan algorithm) |
| BAB 02 | Advanced OS & service fingerprinting (18 probe, nmap-os-db editing) |
| BAB 03 | Advanced NSE Lua programming (coroutine, threading, brute.Driver) |
| BAB 04 | Protocol-level packet crafting (NSE packet lib, Scapy, fragmentation) |
| BAB 05 | Attacking & analyzing encrypted services (TLS/SSL, STARTTLS, JA3) |
| BAB 06 | Advanced Active Directory recon (Kerberos, LDAP, SMB, RPC, WinRM) |
| BAB 07 | Cloud & container scanning (AWS/GCP/Azure, Docker API, Kubernetes) |
| BAB 08 | ICS/SCADA & IoT scanning (Modbus, Siemens S7, BACnet, DNP3, MQTT) |
| BAB 09 | Evading modern EDR, NGFW & IDS (timing, JA3, signature bypass) |
| BAB 10 | Building scan automation pipeline (Python, CI/CD, DefectDojo) |
| BAB 11 | Nmap dalam red team operations (OPSEC, pivot, living off the land) |
| BAB 12 | Performance tuning & internals (libpcap, Masscan pipeline, benchmark) |
| BAB 13 | Threat hunting dengan Nmap — blue team (baseline, ndiff, SIEM) |
| BAB 14 | Building custom vulnerability scanner (NVD API, CVE matcher) |
| BAB 15 | Advanced methodology & master cheatsheet (semua flag, decision tree) |

---

## Jalur Belajar yang Disarankan

```
[Pemula]
   └── beginner/bab01.md → bab15.md  (selesaikan semua)
        └── intermediate/bab01.md → bab15.md  (selesaikan semua)
             └── Advance/bab01.md → bab15.md  (selesaikan semua)

[Sudah tahu dasar Nmap]
   └── Mulai dari intermediate/bab01.md

[Security profesional / pentest experienced]
   └── Langsung ke Advance/bab01.md
```

---

## Tools yang Dibutuhkan

```bash
# Modul Beginner
sudo apt-get install -y nmap zenmap

# Modul Intermediate
sudo apt-get install -y nmap masscan python3 python3-pip
pip3 install python-libnmap

# Modul Advanced
sudo apt-get install -y nmap masscan python3 python3-pip wireshark ldap-utils
pip3 install requests scapy python-libnmap
```

---

## Legal & Etika

> Seluruh materi dalam modul ini HANYA untuk tujuan edukasi, penelitian keamanan yang sah, dan pengujian pada sistem yang **kamu miliki** atau **telah mendapat izin tertulis eksplisit**.

Menggunakan teknik scanning terhadap sistem tanpa izin adalah **tindak pidana** di hampir semua yurisdiksi. Selalu patuhi:
- Scope engagement (SOW / Rules of Engagement)
- Program bug bounty yang berlaku
- Hukum setempat (UU ITE di Indonesia, CFAA di AS, dll.)

---
