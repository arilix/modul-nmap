# BAB 03 — Advanced Target Specification & DNS Enumeration

> **NMAP Intermediate Guide · 2026 Edition**

---

## Target Specification Lanjutan

### Membaca Target dari File

```bash
# Buat file daftar target
cat targets.txt
192.168.1.1
192.168.1.10-20
10.0.0.0/24
example.com

# Scan dari file
nmap -iL targets.txt -sS -T4
```

### Randomisasi Urutan Target

```bash
# Acak urutan host scanning (lebih sulit dideteksi IDS)
nmap --randomize-hosts 192.168.1.0/24

# Kombinasi dengan input file
nmap -iL targets.txt --randomize-hosts -sS -T2
```

### Exclude Host atau Subnet

```bash
# Exclude satu host
nmap 192.168.1.0/24 --exclude 192.168.1.1

# Exclude beberapa host
nmap 192.168.1.0/24 --exclude 192.168.1.1,192.168.1.254

# Exclude dari file
nmap 192.168.1.0/24 --excludefile exclude.txt
```

### Scanning Multiple Subnet Sekaligus

```bash
# Beberapa range sekaligus
nmap 192.168.1.0/24 10.0.0.0/16 172.16.0.0/12

# Kombinasi CIDR, range, dan single IP
nmap 192.168.1.1 192.168.1.100-150 10.0.0.0/24 -sS -T4
```

---

## DNS Enumeration Mendalam

DNS adalah sumber informasi yang sangat kaya tentang infrastruktur target.

### Reverse DNS Lookup

```bash
# Nmap secara default melakukan reverse DNS untuk semua host
nmap -sn 192.168.1.0/24

# Paksa reverse DNS (bahkan untuk host yang tidak merespons)
nmap -sn -R 192.168.1.0/24

# Nonaktifkan DNS resolution (lebih cepat, lebih senyap)
nmap -n 192.168.1.0/24

# Gunakan DNS server spesifik
nmap --dns-servers 8.8.8.8,1.1.1.1 192.168.1.0/24
```

### DNS Bruteforce Subdomain

```bash
# Brute-force subdomain menggunakan NSE
nmap --script dns-brute --script-args dns-brute.domain=example.com

# Dengan wordlist kustom
nmap --script dns-brute \
  --script-args dns-brute.hostlist=/usr/share/wordlists/dns.txt,dns-brute.domain=example.com

# Cek apakah zone transfer diizinkan (misconfiguration serius)
nmap --script dns-zone-transfer --script-args dns-zone-transfer.domain=example.com -p 53 ns1.example.com
```

### SRV Record Enumeration

```bash
# Temukan SRV records (berguna untuk menemukan layanan internal)
nmap --script dns-srv-enum --script-args dns-srv-enum.domain=example.com
```

### Reverse DNS Massal

```bash
# Reverse DNS untuk seluruh /24
nmap -sL 192.168.1.0/24

# Output: daftar semua hostname yang terdaftar
# Berguna untuk mapping jaringan tanpa mengirim paket ke target!
```

> 💡 **Tip:** Flag `-sL` (List Scan) **tidak mengirim paket ke target** sama sekali — hanya melakukan DNS lookup. Ini cara paling aman untuk memulai reconnaissance.

---

## Teknik Resolusi & List Scan untuk Recon Pasif

```bash
# List scan: hanya list target + DNS, tidak ada probing
nmap -sL 10.0.0.0/8 -oN targets_dns.txt

# Hitung berapa host di range tertentu
nmap -sL 192.168.0.0/16 | grep "Nmap scan report" | wc -l
```

---

## Traceroute untuk Network Mapping

```bash
# Tambahkan traceroute ke scan apapun
nmap --traceroute 192.168.1.1

# Traceroute dengan OS detection
sudo nmap -O --traceroute 192.168.1.1

# Traceroute tanpa port scan
nmap -sn --traceroute 192.168.1.0/24
```

**Contoh output traceroute:**
```
TRACEROUTE (using port 80/tcp)
HOP RTT     ADDRESS
1   0.50 ms 192.168.1.1
2   1.20 ms 10.0.0.1
3   5.80 ms 203.0.113.1
```

---

## NSE Scripts untuk Network Enumeration

```bash
# Enumerasi broadcast (temukan host yang tidak merespons ping)
sudo nmap --script broadcast-ping

# CDP neighbor discovery (Cisco devices)
sudo nmap --script cdp-info 192.168.1.0/24

# LLDP neighbor discovery
sudo nmap --script lldp-discovery 192.168.1.0/24

# NetBIOS / Windows hostname enumeration
nmap --script nbstat 192.168.1.0/24

# Whois lookup untuk IP publik
nmap --script whois-ip 203.0.113.1
```

---

## Contoh Workflow Recon Awal

```bash
#!/bin/bash
TARGET="192.168.1.0/24"

echo "[1] List scan + DNS..."
nmap -sL $TARGET -oN 01_list.txt

echo "[2] Host discovery..."
nmap -sn $TARGET -oN 02_hosts.txt

echo "[3] Traceroute..."
nmap -sn --traceroute $TARGET -oN 03_traceroute.txt

echo "[4] DNS brute (jika ada domain)..."
# nmap --script dns-brute --script-args dns-brute.domain=target.com

echo "Done. Hasil di file 01-03."
```

---

*← [BAB 02: Advanced Scan Techniques](bab02.md) · Selanjutnya → [BAB 04: Banner Grabbing & Deep Fingerprinting](bab04.md)*
