# BAB 04 — Banner Grabbing & Deep Fingerprinting

> **NMAP Intermediate Guide · 2026 Edition**

---

## Apa itu Banner Grabbing?

**Banner** adalah teks yang dikirim oleh sebuah layanan saat koneksi pertama kali dibuka. Banner biasanya berisi informasi versi software, nama server, dan kadang OS.

```
SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.6      ← Banner SSH
220 mail.example.com ESMTP Postfix 3.6.4     ← Banner SMTP
Apache/2.4.52 (Ubuntu)                        ← Banner HTTP
```

---

## Banner Grabbing dengan Nmap

### Script `banner`

```bash
# Ambil banner dari semua port terbuka
nmap --script banner 192.168.1.1

# Target port tertentu
nmap --script banner -p 21,22,25,80,110,143 192.168.1.1

# Banner + version detection
nmap -sV --script banner 192.168.1.1
```

### Version Detection Mendalam (`-sV`)

```bash
# Versi dasar
nmap -sV 192.168.1.1

# Intensitas tinggi untuk layanan yang "diam"
nmap -sV --version-intensity 9 192.168.1.1

# Coba semua probe (sangat lambat tapi sangat akurat)
nmap -sV --version-all 192.168.1.1

# Hanya cek satu port, intensitas tinggi
nmap -sV --version-intensity 9 -p 443 192.168.1.1
```

---

## HTTP Fingerprinting Mendalam

### Mengidentifikasi Web Server & Framework

```bash
# Ambil HTTP headers & title
nmap --script http-headers,http-title 192.168.1.1 -p 80,443,8080

# Deteksi teknologi web (CMS, framework)
nmap --script http-generator 192.168.1.1 -p 80

# Cek metode HTTP yang diizinkan
nmap --script http-methods --script-args http-methods.url-path=/api/ -p 80 192.168.1.1

# Temukan file robots.txt
nmap --script http-robots.txt 192.168.1.1

# Enumerate direktori web
nmap --script http-enum 192.168.1.1 -p 80,443

# Cek security headers
nmap --script http-security-headers 192.168.1.1 -p 443
```

**Output contoh `http-enum`:**
```
PORT   STATE SERVICE
80/tcp open  http
| http-enum:
|   /admin/: Possible admin folder
|   /login.php: Possible admin folder
|   /wp-login.php: Wordpress login page
|   /robots.txt: Robots file
|_  /phpinfo.php: Possible information file
```

---

## SSL/TLS Fingerprinting

```bash
# Enum semua cipher suite yang didukung
nmap --script ssl-enum-ciphers -p 443 192.168.1.1

# Cek sertifikat SSL
nmap --script ssl-cert -p 443 192.168.1.1

# Cek kerentanan SSL (Heartbleed, POODLE, dll.)
nmap --script ssl-heartbleed,ssl-poodle,ssl-ccs-injection -p 443 192.168.1.1

# TLS version check
nmap --script ssl-enum-ciphers -p 443 192.168.1.1 | grep -E "TLSv|SSLv"
```

**Output contoh `ssl-cert`:**
```
| ssl-cert: Subject: commonName=example.com/organizationName=Example Inc
| Not valid before: 2025-01-01T00:00:00
| Not valid after:  2026-01-01T00:00:00
|_SHA-1: ab:cd:ef:12:34:56:78:90...
```

---

## SMB Fingerprinting (Windows Networks)

```bash
# Dapatkan info SMB (OS, hostname, domain)
nmap --script smb-os-discovery -p 445 192.168.1.1

# Enum shares SMB
nmap --script smb-enum-shares -p 445 192.168.1.1

# Enum users SMB
nmap --script smb-enum-users -p 445 192.168.1.1

# Cek signing SMB (penting untuk relay attacks)
nmap --script smb-security-mode -p 445 192.168.1.0/24

# Info SMB lengkap
nmap --script smb-os-discovery,smb-enum-shares,smb-security-mode -p 445 192.168.1.1
```

---

## Database Fingerprinting

```bash
# MySQL
nmap --script mysql-info,mysql-empty-password -p 3306 192.168.1.1

# PostgreSQL
nmap --script pgsql-brute -p 5432 192.168.1.1

# MSSQL (SQL Server)
nmap --script ms-sql-info,ms-sql-config -p 1433 192.168.1.1

# MongoDB (tanpa auth)
nmap --script mongodb-info -p 27017 192.168.1.1

# Redis
nmap --script redis-info -p 6379 192.168.1.1
```

---

## SNMP Enumeration

SNMP (Simple Network Management Protocol) sering berjalan di UDP/161 dan menyimpan **banyak sekali informasi** tentang perangkat.

```bash
# Cek apakah SNMP berjalan
sudo nmap -sU -p 161 192.168.1.0/24

# Ambil info SNMP dengan community string default
nmap --script snmp-info -p 161 -sU 192.168.1.1

# Enum system info, interface, routing, user accounts
nmap --script snmp-sysdescr,snmp-interfaces,snmp-netstat -p 161 -sU 192.168.1.1

# Brute-force community string
nmap --script snmp-brute -p 161 -sU 192.168.1.1
```

---

## Fingerprinting IoT & Embedded Devices

```bash
# Telnet banner (banyak IoT masih pakai telnet)
nmap --script telnet-ntlm-info,banner -p 23 192.168.1.0/24

# UPnP device info
nmap --script upnp-info -p 1900 -sU 192.168.1.0/24

# Printer enumeration
nmap --script printer-info -p 9100 192.168.1.0/24
```

---

## Tips: Menyaring Output Version Detection

```bash
# Hanya tampilkan baris yang berisi versi
nmap -sV 192.168.1.0/24 | grep -E "open.*[0-9]+\.[0-9]+"

# Simpan dan filter
nmap -sV -oN scan.txt 192.168.1.0/24
grep "open" scan.txt | awk '{print $1, $3, $4, $5}'
```

---

*← [BAB 03: Advanced Target & DNS Enum](bab03.md) · Selanjutnya → [BAB 05: Advanced NSE Script Categories](bab05.md)*
