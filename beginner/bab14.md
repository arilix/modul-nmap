# BAB 14 — Real-World Use Cases

> **NMAP Guide for Beginners · 2026 Edition · Written by Rocky**

---

## Network Inventory & Auditing

```bash
# Temukan semua host aktif di jaringan
nmap -sn 192.168.1.0/24 -oN live_hosts.txt

# Full inventory scan
sudo nmap -sS -sV -O -T4 192.168.1.0/24 -oA network_audit

# Temukan semua web server
nmap -p 80,443,8080,8443 --open 192.168.1.0/24

# Temukan semua SSH server
nmap -p 22 --open 192.168.1.0/24
```

---

## Vulnerability Assessment

```bash
# Cek kerentanan umum
sudo nmap --script=vuln 192.168.1.1

# Cek EternalBlue (MS17-010 / WannaCry)
nmap --script smb-vuln-ms17-010 -p445 192.168.1.1

# Cek masalah SSL/TLS
nmap --script ssl-enum-ciphers -p 443 192.168.1.1

# Cek HTTP security headers
nmap --script http-security-headers -p 80,443 192.168.1.1

# Cek FTP anonymous login
nmap --script ftp-anon -p 21 192.168.1.0/24
```

---

## Service Enumeration

```bash
# Database servers
nmap -p 1433,3306,5432,27017 --open 192.168.1.0/24

# SMB / Windows shares
nmap -p 445 --script smb-enum-shares 192.168.1.1

# SNMP enumeration
sudo nmap -sU -p 161 --script snmp-info 192.168.1.0/24

# DNS zone transfer attempt
nmap --script dns-zone-transfer -p 53 192.168.1.1
```

---

> ⚠️ **Ingat:** Semua teknik di atas hanya boleh digunakan pada jaringan yang kamu miliki atau telah mendapat izin tertulis dari pemiliknya. Penggunaan tanpa izin adalah ilegal dan tidak etis.

---

*← [BAB 13: Zenmap GUI](bab13.md) · Selanjutnya → [BAB 15: Cheat Sheet & Penutup](bab15.md)*
