# BAB 05 — Advanced NSE Script Categories & Script Arguments

> **NMAP Intermediate Guide · 2026 Edition**

---

## Memahami NSE Lebih Dalam

NSE scripts ditulis dalam **Lua** dan berjalan di dalam Nmap engine. Setiap script mendefinisikan:
- `categories{}` — kategori script
- `portrule` / `hostrule` / `prerule` / `postrule` — kapan script dijalankan
- `action()` — fungsi utama yang dieksekusi

---

## Cara Menemukan & Membaca Script

```bash
# List semua script
ls /usr/share/nmap/scripts/

# Cari script berdasarkan kata kunci
ls /usr/share/nmap/scripts/ | grep "http"
ls /usr/share/nmap/scripts/ | grep "smb"
ls /usr/share/nmap/scripts/ | grep "vuln"

# Baca dokumentasi script langsung dari Nmap
nmap --script-help http-enum
nmap --script-help "http-*"
nmap --script-help all | grep -A5 "ssl-heartbleed"
```

---

## Menjalankan Script Berdasarkan Kategori

```bash
# Kategori default (aman, informatif)
nmap -sC 192.168.1.1           # sama dengan --script=default

# Kategori discovery
nmap --script discovery 192.168.1.1

# Kategori vuln (cek kerentanan)
sudo nmap --script vuln 192.168.1.1

# Kategori safe (tidak merusak)
nmap --script safe 192.168.1.1

# Gabungkan beberapa kategori
nmap --script "default and safe" 192.168.1.1
nmap --script "discovery,version" 192.168.1.1

# Kecualikan kategori tertentu
nmap --script "not intrusive" 192.168.1.1
nmap --script "default and not brute" 192.168.1.1
```

---

## Boolean Expressions dalam Pemilihan Script

Nmap mendukung ekspresi boolean untuk memilih script secara presisi:

```bash
# Semua script kecuali yang brute-force
nmap --script "not brute" 192.168.1.1

# Semua http scripts yang bukan brute
nmap --script "http-* and not http-brute" 192.168.1.1 -p 80

# Script vuln ATAU exploit
nmap --script "vuln or exploit" 192.168.1.1

# Script untuk auth testing yang safe
nmap --script "auth and safe" 192.168.1.1
```

---

## Script Arguments (`--script-args`)

Banyak script yang membutuhkan atau bisa dikustomisasi dengan argumen.

### Format

```bash
nmap --script <name> --script-args key=value,key2=value2
```

### HTTP Scripts

```bash
# Enum dengan path spesifik
nmap --script http-enum \
  --script-args http-enum.basepath=/app/ \
  -p 80 192.168.1.1

# HTTP Auth brute
nmap --script http-brute \
  --script-args brute.mode=user,\
                userdb=/usr/share/wordlists/users.txt,\
                passdb=/usr/share/wordlists/passwords.txt \
  -p 80 192.168.1.1

# HTTP PUT method test
nmap --script http-put \
  --script-args http-put.url=/uploads/test.txt,http-put.file=/tmp/test.txt \
  -p 80 192.168.1.1
```

### SMB Scripts

```bash
# Enum shares dengan credentials
nmap --script smb-enum-shares \
  --script-args smbusername=admin,smbpassword=password \
  -p 445 192.168.1.1

# Enum users with domain
nmap --script smb-enum-users \
  --script-args smbdomain=WORKGROUP \
  -p 445 192.168.1.1
```

### Brute-Force Scripts

```bash
# SSH brute dengan wordlist custom
nmap --script ssh-brute \
  --script-args userdb=users.txt,passdb=pass.txt,brute.firstonly=true \
  -p 22 192.168.1.1

# FTP brute
nmap --script ftp-brute \
  --script-args userdb=users.txt,passdb=pass.txt \
  -p 21 192.168.1.1

# Limit percobaan (jangan lockout account)
nmap --script ssh-brute \
  --script-args brute.delay=2s,brute.limit=10 \
  -p 22 192.168.1.1
```

---

## Script Arguments Global

```bash
# Timeout koneksi untuk semua scripts
nmap --script-args newtargets,timeout=30s 192.168.1.1

# Output verbose dari script
nmap -sC -v 192.168.1.1

# Debug script (sangat verbose)
nmap --script http-title -d3 192.168.1.1
```

---

## Script Debugging

```bash
# Level debug 1 (ringan)
nmap --script http-enum -d 192.168.1.1

# Level debug 3 (verbose Lua trace)
nmap --script http-enum -d3 192.168.1.1

# Tampilkan semua output script meskipun kosong
nmap --script http-enum --script-trace 192.168.1.1
```

---

## Script Favorit per Protokol

### Web / HTTP
```bash
nmap --script "http-title,http-headers,http-methods,http-auth-finder,http-enum,http-security-headers" \
  -p 80,443,8080,8443 192.168.1.1
```

### SSH
```bash
nmap --script "ssh-hostkey,ssh-auth-methods,ssh2-enum-algos" \
  -p 22 192.168.1.1
```

### Email (SMTP/POP3/IMAP)
```bash
nmap --script "smtp-commands,smtp-open-relay,pop3-capabilities,imap-capabilities" \
  -p 25,110,143,465,587,993,995 192.168.1.1
```

### Windows / Active Directory
```bash
nmap --script "smb-os-discovery,smb-security-mode,smb-enum-shares,\
  smb-vuln-ms17-010,ms-sql-info,ldap-rootdse" \
  -p 139,445,1433,389 192.168.1.1
```

---

## Update Script Database

```bash
# Setelah install/update Nmap, update database script
sudo nmap --script-updatedb

# Cek tanggal update terakhir
ls -la /usr/share/nmap/scripts/*.nse | tail -5
```

---

*← [BAB 04: Banner Grabbing & Fingerprinting](bab04.md) · Selanjutnya → [BAB 06: Menulis Custom NSE Script](bab06.md)*
