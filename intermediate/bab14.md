# BAB 14 — CTF & Pentest Scenarios

> **NMAP Intermediate Guide · 2026 Edition**

---

## Skenario 1: CTF — Menemukan Port Tersembunyi

Port CTF sering tidak berada di port standar. Strategi:

```bash
# Scan SEMUA 65535 port (wajib di CTF)
sudo nmap -p- -sS -T4 10.10.10.100

# Versi lebih cepat dengan min-rate
sudo nmap -p- -sS --min-rate 5000 -T4 10.10.10.100

# Full combo CTF
sudo nmap -p- -sS -sV -sC --open -T4 10.10.10.100 -oA ctf_scan

# UDP juga (sering dilupakan)
sudo nmap -sU --top-ports 200 10.10.10.100
```

### Setelah Nemukan Port: Version Hunting

```bash
# Port tidak dikenal? Coba grab banner
nmap --script banner -p <port> 10.10.10.100

# Intensitas versi maksimum
nmap -sV --version-all -p <port> 10.10.10.100

# Coba koneksi manual
nc -nv 10.10.10.100 <port>
telnet 10.10.10.100 <port>
curl -v http://10.10.10.100:<port>
```

---

## Skenario 2: CTF — Web Server Enumeration

```bash
# Langkah pertama: cek semua port HTTP/HTTPS
sudo nmap -p 80,443,8000,8080,8443,8888,9000,9090 --open -sV 10.10.10.100

# Enumerate direktori, file, dan teknologi
nmap --script "http-enum,http-title,http-headers,http-methods,http-robots.txt" \
  -p 80,8080 10.10.10.100

# Cari phpinfo, backup files, git repos
nmap --script http-enum \
  --script-args http-enum.basepath='/' \
  -p 80 10.10.10.100

# Coba semua vhost dengan wordlist
nmap --script http-vhosts -p 80 10.10.10.100
```

---

## Skenario 3: CTF — Menemukan Credential Default

```bash
# FTP anonymous
nmap --script ftp-anon -p 21 10.10.10.100

# MySQL tanpa password
nmap --script mysql-empty-password -p 3306 10.10.10.100

# Telnet
nmap --script telnet-ntlm-info -p 23 10.10.10.100

# Redis tanpa auth
nmap --script redis-info -p 6379 10.10.10.100

# MongoDB tanpa auth
nmap --script mongodb-info -p 27017 10.10.10.100

# SNMP dengan community string "public"
nmap --script snmp-info -sU -p 161 10.10.10.100
```

---

## Skenario 4: Pentest Internal — Active Directory

```bash
# Temukan domain controller (port khas: 88, 389, 445, 636, 3268)
sudo nmap -p 88,389,445,636,3268,3269 --open 192.168.1.0/24

# Identifikasi DC dan info domain
nmap --script "ldap-rootdse,smb-os-discovery,ms-sql-info" \
  -p 389,445,1433 192.168.1.0/24

# Kerberoasting preparation: temukan SPN
nmap --script krb5-enum-users -p 88 192.168.1.0/24

# Cek LDAP signing (misconfiguration)
nmap --script ldap-search \
  --script-args ldap.username='',ldap.password='' \
  -p 389 192.168.1.1
```

---

## Skenario 5: Pentest Eksternal — Recon Perimeter

```bash
# Langkah 1: Resolusi domain target
nmap -sL target.com       # List scan + DNS
nmap --script dns-brute --script-args dns-brute.domain=target.com

# Langkah 2: Scan perimeter umum
sudo nmap -sS -sV -T4 --open \
  -p 21,22,25,53,80,110,443,465,587,993,995,3389,8080,8443 \
  target.com

# Langkah 3: SSL/TLS audit
nmap --script ssl-enum-ciphers,ssl-cert,ssl-heartbleed -p 443 target.com

# Langkah 4: Email security check
nmap --script smtp-open-relay,smtp-commands -p 25,587 target.com

# Langkah 5: WAF detection
nmap --script http-waf-detect -p 80,443 target.com
```

---

## Skenario 6: Post-Exploitation — Network Pivot Discovery

Setelah mendapatkan akses ke satu host, gunakan Nmap dari host tersebut untuk recon jaringan internal:

```bash
# Upload Nmap ke host yang sudah dikompromis (atau gunakan yang ada)
# Scan jaringan internal dari dalam

# Discovery cepat
nmap -sn 10.0.0.0/24 --min-parallelism 50

# Cari host dengan layanan menarik
nmap -p 22,3389,445,80,443 --open 10.0.0.0/24

# Gunakan --proxies untuk scan melalui SOCKS proxy
nmap --proxies socks4://127.0.0.1:1080 10.0.0.0/24
```

---

## Template Scan untuk Berbagai Skenario

### Quick & Dirty (CTF awal)
```bash
sudo nmap -p- --min-rate 5000 -sV -sC --open -T4 <IP> -oA full
```

### Comprehensive Internal Pentest
```bash
sudo nmap -sS -sU -T4 -A -v -PE -PP -PS80,443 -PA3389 \
  --script "default and safe" \
  192.168.1.0/24 \
  -oA comprehensive_scan
```

### Stealth & Slow (evade IDS)
```bash
sudo nmap -sS -T1 -D RND:10 -f --data-length 20 \
  --randomize-hosts \
  -p 22,80,443,445,3389 \
  192.168.1.0/24 \
  -oA stealth_scan
```

### Web Application Focus
```bash
sudo nmap -sS -sV -T4 \
  --script "http-*" \
  -p 80,443,8080,8443,8000,8888 \
  --open \
  192.168.1.0/24 \
  -oA web_scan
```

---

## Tips CTF dengan Nmap

| Situasi                    | Command yang Tepat                                    |
|----------------------------|-------------------------------------------------------|
| Port tidak ditemukan        | `nmap -p-` (scan semua port)                         |
| Service tidak dikenal       | `--script banner` + `nc -nv`                         |
| Web 403/401                 | `http-auth-finder`, coba semua method HTTP           |
| Windows domain              | `smb-os-discovery`, cek port 88 (Kerberos)           |
| Banyak port filtered        | `-Pn`, coba `-sA` untuk peta firewall                |
| Version tidak terdeteksi    | `--version-all --version-intensity 9`                 |
| UDP service                 | `-sU`, fokus 53,67,68,69,123,161,500,4500            |

---

*← [BAB 13: Nmap + Tools Lain](bab13.md) · Selanjutnya → [BAB 15: Metodologi Recon & Advanced Cheatsheet](bab15.md)*
