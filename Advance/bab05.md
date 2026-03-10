# BAB 05 — Attacking & Analyzing Encrypted Services

> **NMAP Advanced Guide · 2026 Edition**

---

## TLS/SSL: Lebih dari Sekadar Verifikasi Sertifikat

Layanan terenkripsi bukan berarti tidak bisa dianalisis. Nmap memiliki kapabilitas mendalam untuk:
- Enumerasi cipher suite dan protokol yang didukung
- Deteksi kerentanan SSL/TLS klasik dan modern
- Analisis sertifikat (expiry, CA, SAN headers)
- STARTTLS enumeration pada protokol non-HTTP

---

## NSE Scripts untuk SSL/TLS Analysis

### ssl-cert — Ekstrak Detail Sertifikat

```bash
# Informasi sertifikat lengkap
nmap --script ssl-cert -p 443 target.com

# Output:
# | ssl-cert: Subject: commonName=target.com/organizationName=Corp
# | Subject Alternative Name: DNS:target.com, DNS:www.target.com
# | Not valid before: 2024-01-01T00:00:00
# | Not valid after:  2025-01-01T23:59:59
# | MD5:   aa:bb:cc:dd...
# |_SHA-1: 11:22:33:44...
```

### ssl-enum-ciphers — Audit Cipher Suite

```bash
# Enumerasi semua cipher suite yang diterima server
nmap --script ssl-enum-ciphers -p 443 target.com

# Output (dengan grading A/B/C/F):
# | ssl-enum-ciphers:
# |   TLSv1.2:
# |     ciphers:
# |       TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (ecdh_x25519) - A
# |       TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (ecdh_x25519) - A
# |       TLS_RSA_WITH_RC4_128_SHA - F  (weak!)
# |   TLSv1.0:
# |     ciphers:
# |       TLS_RSA_WITH_DES_CBC_SHA - F  (very weak!)
```

```bash
# Scan multiple ports sekaligus
nmap --script ssl-enum-ciphers -p 443,8443,9443 192.168.1.0/24

# Scan dengan filter grading buruk saja
nmap --script ssl-enum-ciphers -p 443 target.com 2>/dev/null | grep " - [BCDF]"
```

---

## Deteksi Kerentanan SSL/TLS Klasik

```bash
# POODLE (SSLv3 downgrade) - CVE-2014-3566
nmap --script ssl-poodle -p 443 target.com

# DROWN (SSLv2 pada server berbagi key) - CVE-2016-0800
nmap --script sslv2-drown -p 443 target.com

# HEARTBLEED (OpenSSL memory leak) - CVE-2014-0160
nmap --script ssl-heartbleed -p 443 target.com

# BEAST (CBC mode IV attack) - CVE-2011-3389
nmap --script ssl-enum-ciphers -p 443 target.com | grep "CBC.*TLSv1\.0"

# LOGJAM (weak DH params) - CVE-2015-4000
nmap --script ssl-dh-params -p 443 target.com

# FREAK (export-grade RSA) - CVE-2015-0204
nmap --script ssl-enum-ciphers -p 443 target.com | grep "EXPORT"
```

### CCS Injection (OpenSSL ChangeCipherSpec)

```bash
nmap --script ssl-ccs-injection -p 443 target.com

# Output jika vulnerable:
# | ssl-ccs-injection:
# |   VULNERABLE:
# |   SSL/TLS MITM vulnerability (CCS Injection)
# |     State: VULNERABLE
```

---

## STARTTLS Enumeration

STARTTLS memungkinkan protokol plaintext untuk upgrade ke TLS. Banyak sysadmin lupa mengaudit ini.

```bash
# SMTP STARTTLS
nmap --script smtp-starttls-check -p 25,587 target.com
nmap --script ssl-enum-ciphers -p 25 --script-args tls.servername=mail.target.com target.com

# IMAP STARTTLS
nmap --script imap-capabilities -p 143 target.com  # Cek apakah STARTTLS ada
nmap --script ssl-enum-ciphers -p 143 target.com

# POP3 STARTTLS
nmap --script ssl-enum-ciphers -p 110 target.com

# FTP STARTTLS (AUTH TLS/SSL)
nmap --script ftp-anon,ssl-enum-ciphers -p 21 target.com

# LDAP STARTTLS
nmap --script ldap-rootdse -p 389 target.com  # lihat apakah supportedExtension ada
nmap --script ssl-enum-ciphers -p 389 target.com

# XMPP STARTTLS
nmap --script ssl-enum-ciphers -p 5222 target.com
```

---

## TLS Fingerprinting dengan JA3

JA3 adalah fingerprint dari ClientHello TLS — berguna untuk deteksi tools/scanner.

```bash
# Capture TLS ClientHello yang dikirim Nmap
sudo tcpdump -i eth0 -w /tmp/tls_capture.pcap "host target.com and port 443" &
nmap --script ssl-enum-ciphers -p 443 target.com
kill %1

# Ekstrak JA3 fingerprint dari capture
ja3 /tmp/tls_capture.pcap
# atau
python3 -m ja3 /tmp/tls_capture.pcap

# Nmap memiliki JA3 fingerprint yang diketahui:
# Lookupable di: https://ja3er.com/search/{hash}
```

### Menghindari Deteksi JA3

```bash
# Gunakan --proxies untuk akses melalui proxy (ubah TLS stack)
nmap --proxies http://127.0.0.1:8080 --script ssl-enum-ciphers -p 443 target.com

# Modifikasi TLS di level NSE dengan script-args
nmap --script ssl-enum-ciphers \
     --script-args ssl.version=tlsv1_2 \
     -p 443 target.com
```

---

## Analisis Sertifikat: Advanced Use Cases

### Ekstrak SANs untuk Subdomain Discovery

```bash
# Subject Alternative Names sering mengungkap subdomain tersembunyi
nmap --script ssl-cert -p 443 target.com | grep "DNS:"

# Automasi dengan bash
for ip in $(cat hosts.txt); do
    nmap --script ssl-cert -p 443 $ip 2>/dev/null |
    grep -oP 'DNS:[^,\s]+' | sed 's/DNS://'
done | sort -u > discovered_domains.txt
```

### Certificate Transparency Log Correlation

```bash
# Bandingkan SAN dari scan dengan CT logs
# Tools: crt.sh, censys, cert-spotter

# Cari wildcard certificates yang expose internal names
nmap --script ssl-cert -p 443 192.168.1.0/24 2>/dev/null |
grep -B5 "\*\." | grep "commonName\|Subject Alt"
```

---

## mTLS (Mutual TLS) Service Detection

```bash
# Deteksi apakah server meminta client certificate
nmap --script ssl-cert -p 443 target.com 2>&1 | grep -i "client"

# Python untuk test mTLS requirement
python3 << 'EOF'
import ssl
import socket

ctx = ssl.create_default_context()
try:
    with ctx.wrap_socket(socket.socket(), server_hostname="target.com") as s:
        s.connect(("target.com", 443))
        print("No mTLS required")
except ssl.SSLError as e:
    if "certificate required" in str(e).lower():
        print("mTLS DETECTED - server requires client certificate")
    else:
        print(f"SSL Error: {e}")
EOF
```

---

## SSH Deep Analysis

```bash
# Enumerasi host keys dan algoritma SSH
nmap --script ssh-hostkey,ssh2-enum-algos -p 22 target.com

# Output:
# | ssh2-enum-algos:
# |   kex_algorithms: (9)
# |       curve25519-sha256
# |       diffie-hellman-group14-sha256  <-- acceptable
# |       diffie-hellman-group1-sha1     <-- WEAK (CVE-2016-0777)
# |   server_host_key_algorithms:
# |       rsa-sha2-512, rsa-sha2-256, ssh-rsa
# |   encryption_algorithms:
# |       aes128-ctr, aes192-ctr, aes256-ctr
# |       arcfour256                     <-- WEAK (RC4)

# Deteksi SSH version untuk CVE mapping
nmap -sV -p 22 target.com | grep "OpenSSH"
# OpenSSH < 7.7 - User enumeration (CVE-2018-15473)
# OpenSSH < 8.5 - Agent forwarding issues

# SSH user enumeration (CVE-2018-15473)
nmap --script ssh-auth-methods \
     --script-args="ssh.user=root" \
     -p 22 target.com
```

---

## RDP TLS Analysis

```bash
# RDP certificate dan cipher analysis
nmap --script rdp-enum-encryption -p 3389 target.com

# Output:
# | rdp-enum-encryption:
# |   Security layer
# |     CredSSP (NLA): SUCCESS (Message sent)
# |   RDP Encryption level: Client Compatible
# |   RDP Protocol version: RDP 5.x, 6.x, 7.x, 8.x, 10.x

# Cek apakah BlueKeep atau DejaBlue vulnerable
nmap --script rdp-vuln-ms12-020 -p 3389 target.com
```

---

## IPSec / VPN Fingerprinting

```bash
# IKEv1/IKEv2 (ISAKMP) enumeration - port 500/udp, 4500/udp
nmap -sU --script ike-version -p 500,4500 target.com

# Output:
# | ike-version:
# |   vendor_id: Microsoft Windows 8 (or Server 2012)
# |   attributes: (3)
# |       AES-CBC-256/SHA1/DH-14
# |       AES-CBC-128/SHA256/DH-2
# |       3DES-CBC/MD5/DH-1    <-- WEAK

# SSL VPN fingerprinting (OpenVPN)
nmap -sU -p 1194 target.com --script openvpn-info
```

---

*← [BAB 04: Packet Crafting](bab04.md) · Selanjutnya → [BAB 06: Active Directory Recon](bab06.md)*
