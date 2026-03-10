# BAB 02 — Advanced OS & Service Fingerprinting

> **NMAP Advanced Guide · 2026 Edition**

---

## Bagaimana OS Fingerprinting Benar-Benar Bekerja

Nmap menggunakan **18 probe TCP/UDP/ICMP berbeda** dan menganalisis respons untuk membentuk "fingerprint" yang dibandingkan dengan database `nmap-os-db`.

### 18 Probe yang Dikirim

```
TCP Probes (7):
  SEQ1-SEQ6 → Kirim ke port OPEN: berbeda window size, options, flags
  ECN        → SYN + ECN flags

TCP Probes (3) ke port CLOSED:
  T2  → SYN, window=128, DF set
  T3  → SYN|FIN|URG|PSH, window=256, no DF
  ...

UDP Probe (1):
  Ke UDP port tertutup → trigger ICMP Port Unreachable

ICMP Probes (2):
  IE1, IE2 → Echo request dengan berbagai parameter

TCP Window/Option Probes:
  T4-T7 → Ke port closed
```

### Parameter yang Dianalisis

```
Probe → Respons → Extract nilai:

SEQ: (IP ID sequence, ISN, timestamp)  
OPS: (TCP options: MSS, SACK, WS, NOP, TS)  
WIN: (TCP window size per probe)
ECN: (ECN support)
T1-T7: (flag combinations, DF bit, TTL)
U1: (ICMP unreachable response structure)
IE: (ICMP echo response: IP DF, TTL, ID, Response)
```

---

## Membaca Format Database OS Fingerprint

```bash
# Buka database
less /usr/share/nmap/nmap-os-db

# Contoh entry:
# Fingerprint Linux 5.15 - 6.1 (Ubuntu/Debian)
# Class Linux | Linux | 5.X | general purpose
# CPE cpe:/o:linux:linux_kernel:5
# SEQ(SP=FA-104%GCD=1-6%ISR=108-112%TI=Z%CI=Z%II=I%TS=A)
# OPS(O1=M5B4ST11NW7%O2=M5B4ST11NW7%O3=M5B4NNT11NW7%O4=M5B4ST11NW7%O5=M5B4ST11NW7%O6=M5B4ST11)
# ...
```

### Interpretasi Field

```
SP    = Scan Port (sequence predictability)
GCD   = Greatest Common Divisor (ISN analysis)
ISR   = ISN Rate (sequence increment rate)
TI    = IP ID sequence type: Z=zero, RD=random, RI=random+increment
NW7   = TCP timestamp option window scaling
T     = TCP timestamp (A=active, U=unsupported)
```

---

## Membuat OS Fingerprint Baru

```bash
# Scan target dengan verbose OS detection
sudo nmap -O --osscan-guess -v 192.168.1.1 2>&1 | grep -A 50 "OS fingerprint"

# Output akan menampilkan fingerprint yang tidak dikenal:
# OS fingerprint not in Nmap OS database. Please submit at
# https://nmap.org/submit/
# OS:SCAN(V=7.95%E=4%D=3/10%OT=22%CT=1%CU=30413%PV=Y%DS=1%DC=D%G=Y%TM=...
```

### Menambahkan ke Database Lokal

```bash
# Edit /usr/share/nmap/nmap-os-db
sudo nano /usr/share/nmap/nmap-os-db

# Tambahkan entry baru dengan format:
# Fingerprint Custom Device XYZ
# Class vendor | OS | version | type
# CPE cpe:/o:linux:linux_kernel:5
# SEQ(...) OPS(...) WIN(...) ECN(...) T1(...) ...
```

---

## Service Probe Database: Cara Kerjanya

`/usr/share/nmap/nmap-service-probes` berisi probe dan match pattern untuk version detection.

### Struktur File

```
# Probe NULL: koneksi tanpa data → tunggu banner
Probe TCP NULL q||
rarity 1
ports 21,22,25,80,...

# Match pattern (regex)
match ftp m|^220[\- ](.*)\r\n| p/FTP/ v/$1/ i/FTP server/

# Softmatch: partial match, mungkin benar
softmatch ftp m|^220 |

# Probe HTTP: kirim HTTP GET
Probe TCP GetRequest q|GET / HTTP/1.0\r\n\r\n|
rarity 2
ports 80,8080,...
sslports 443,8443,...

match http m|^HTTP/1\..* \d\d\d .*\r\nServer: (Apache[^\r\n]*)\r\n| p/Apache httpd/ v/$1/ i/server/
```

### Menambahkan Match Pattern Custom

```bash
# Contoh: tambahkan deteksi untuk service proprietary
sudo nano /usr/share/nmap/nmap-service-probes

# Tambahkan di section yang tepat:
match myapp m|^MYAPP/(\d+\.\d+) Ready| p/MyApp Server/ v/$1/
```

---

## OS Detection untuk Perangkat Non-Standar

### Fingerprinting Embedded Devices

```bash
# Device embedded sering tidak match OS database
sudo nmap -O --osscan-guess --max-os-tries 5 192.168.1.1

# Force OS scan meskipun kondisi tidak ideal
sudo nmap -O --osscan-limit 192.168.1.0/24

# Kombinasi dengan version untuk konteks lebih baik
sudo nmap -O -sV -T4 192.168.1.1
```

### TCP/IP Stack Analysis Manual

```bash
# Kirim SYN ke port tertutup, analisis TTL dan Window
sudo nmap --packet-trace -sS -p 1 192.168.1.1 2>&1 | grep -E "SENT|RCVD"

# Interpretasi TTL:
# TTL 64  → Linux/macOS (default)
# TTL 128 → Windows (default)
# TTL 255 → Network device / Cisco IOS
# TTL 254 → Cisco device
# TTL 30  → Juniper / HP
```

---

## Version Detection: Regex Lanjutan

NSE dan nmap-service-probes menggunakan PCRE regex yang powerful:

```bash
# Option p/ = product name
# Option v/ = version
# Option i/ = info
# Option h/ = hostname
# Option o/ = OS
# Option d/ = device type
# Option cpe:/ = CPE string

# Contoh match lengkap:
match ssh m|^SSH-(\d+\.\d+)-OpenSSH[_\- ](\S+)[\r\n]| 
  p/OpenSSH/ v/$2/ i/protocol $1/ 
  cpe:/a:openbsd:openssh:$2/
```

### Debug Version Detection

```bash
# Lihat setiap probe yang dikirim dan match yang dicoba
sudo nmap --version-trace -sV -p 22 192.168.1.1

# Output menampilkan:
# Probe: NULL -> "SSH-2.0-OpenSSH_8.9\r\n"
# Matched: OpenSSH 8.9 (protocol 2.0)
```

---

## HTTP Application Fingerprinting

### Passive Fingerprinting dari Headers

```bash
# Ambil semua header
nmap --script http-headers -p 80,443 192.168.1.1

# Identifikasi WAF dari headers
nmap --script http-waf-detect,http-waf-fingerprint -p 80,443 192.168.1.1

# CMS detection
nmap --script http-generator,http-cms-detect -p 80 192.168.1.1
```

### Active Fingerprinting: Error Pages & Behavior

```bash
# Analisis error page untuk identifikasi framework
nmap --script http-errors -p 80 192.168.1.1

# Detect server dari 404 behavior
nmap --script http-404-brute -p 80 192.168.1.1

# Test semua method HTTP untuk anomali
nmap --script http-methods \
  --script-args http-methods.test-all=true \
  -p 80,443 192.168.1.1
```

---

## Menulis Probe Version Detection Sendiri

```bash
# Tambahkan ke nmap-service-probes:

# Probe untuk service custom di port 9999
Probe TCP MyProbe q|HELLO\r\n|
rarity 7
ports 9999

# Match response: "MYSERVICE/1.2.3 Ready"
match myservice m|^MYSERVICE/(\d+\.\d+\.\d+) (.+)\r\n|
  p/MyService/ v/$1/ i/$2/
  cpe:/a:vendor:myservice:$1/

# Softmatch jika hanya sebagian match
softmatch myservice m|^MYSERVICE|
```

---

## Fingerprint Evasion: Bagaimana Target Menyembunyikan Diri

Banyak server modern menyembunyikan versi. Ini cara mendeteksinya:

```bash
# Server menyembunyikan versi Apache:
# Server: Apache  (tanpa versi)
# Cara bypass: cek error pages

nmap --script http-server-header -p 80 192.168.1.1

# Atau trigger error yang mengungkap versi
# Kirim request malformed
nmap --script http-method-tamper -p 80 192.168.1.1

# Cek X-Powered-By header
nmap --script http-headers -p 80 192.168.1.1 | grep -i "powered\|generator\|framework"
```

---

*← [BAB 01: Nmap Architecture](bab01.md) · Selanjutnya → [BAB 03: Advanced NSE Lua Programming](bab03.md)*
