# BAB 02 — Advanced Scan Techniques

> **NMAP Intermediate Guide · 2026 Edition**

---

## Idle Scan / Zombie Scan (`-sI`)

Salah satu teknik paling canggih di Nmap. Idle scan memungkinkan kamu memindai target **tanpa mengirimkan satu pun paket dengan IP aslimu** — seluruh scan dilakukan via host "zombie" di jaringan.

### Cara Kerja

Teknik ini memanfaatkan **IP ID field** di header IP. Setiap paket yang dikirim host akan menginkrementasi IP ID secara berurutan.

```
Langkah 1: Probe zombie → dapatkan IP ID awal (misal: 100)
Langkah 2: Kirim SYN ke target dengan source IP = zombie
Langkah 3: Jika port target OPEN  → target kirim SYN-ACK ke zombie
                                  → zombie balas RST → IP ID naik ke 102
           Jika port target CLOSED → target kirim RST ke zombie
                                   → zombie tidak bereaksi → IP ID tetap 101
Langkah 4: Probe zombie kembali → bandingkan IP ID
           ID naik 2 = port OPEN
           ID naik 1 = port CLOSED/FILTERED
```

### Syarat Zombie yang Baik

- Traffic rendah (IP ID incrementnya predictable)
- Tidak menggunakan IP ID acak (banyak OS modern randomize IP ID!)

```bash
# Cari zombie yang cocok dengan script ipidseq
nmap --script ipidseq 192.168.1.0/24

# Jalankan idle scan menggunakan zombie
sudo nmap -sI 192.168.1.50 192.168.1.1

# Idle scan port spesifik
sudo nmap -sI 192.168.1.50:80 192.168.1.1 -p 22,80,443
```

> ⚠️ Idle scan sangat lambat dan tidak selalu berhasil di jaringan modern karena banyak OS sekarang mengacak IP ID.

---

## IP Protocol Scan (`-sO`)

Bukan memindai port, melainkan memindai **protokol IP mana yang didukung** oleh target (TCP=6, UDP=17, ICMP=1, OSPF=89, dst).

```bash
sudo nmap -sO 192.168.1.1
```

**Contoh output:**
```
PROTOCOL  STATE         SERVICE
1         open          icmp
6         open          tcp
17        open          udp
89        open|filtered ospf
```

> Berguna dalam network auditing untuk menemukan protokol routing (OSPF, EIGRP) yang diekspos.

---

## SCTP Scanning

**SCTP (Stream Control Transmission Protocol)** digunakan di jaringan telekomunikasi (SS7, VoIP). Nmap mendukung dua tipe SCTP scan:

```bash
# SCTP INIT Scan (seperti SYN scan untuk TCP)
sudo nmap -sY 192.168.1.1

# SCTP COOKIE-ECHO Scan (lebih stealth dari INIT)
sudo nmap -sZ 192.168.1.1
```

| Tipe       | Flag  | State bila Open     | Cocok untuk                     |
|------------|-------|---------------------|---------------------------------|
| INIT       | `-sY` | INIT-ACK diterima   | Scan umum SCTP                  |
| COOKIE-ECHO| `-sZ` | Tidak ada RST       | Menghindari beberapa firewall   |

---

## FTP Bounce Scan (`-b`)

Teknik lama (legacy) yang memanfaatkan bug di FTP server lama. FTP server digunakan sebagai proxy untuk memindai target lain.

```bash
nmap -b ftp-server.victim.com 192.168.1.1
```

> ⚠️ Hampir tidak ada FTP server modern yang masih rentan. Berguna untuk historical knowledge dan pentest jaringan lama.

---

## Membandingkan Semua Scan Types

| Scan Type         | Flag   | Root? | Stealth | Reliability | Use Case                       |
|-------------------|--------|-------|---------|-------------|--------------------------------|
| TCP SYN           | `-sS`  | Yes   | Medium  | High        | Default, paling umum           |
| TCP Connect       | `-sT`  | No    | Low     | High        | Tanpa root privilege           |
| UDP               | `-sU`  | Yes   | Medium  | Low-Med     | DNS, SNMP, DHCP                |
| TCP Null          | `-sN`  | Yes   | High    | Low         | Evade non-stateful firewall    |
| TCP FIN           | `-sF`  | Yes   | High    | Low         | Evade non-stateful firewall    |
| TCP Xmas          | `-sX`  | Yes   | High    | Low         | Evade non-stateful firewall    |
| TCP ACK           | `-sA`  | Yes   | Medium  | N/A*        | Pemetaan firewall rules        |
| TCP Window        | `-sW`  | Yes   | Medium  | Low         | Varian ACK scan                |
| TCP Maimon        | `-sM`  | Yes   | High    | Very Low    | OS tertentu saja               |
| Idle/Zombie       | `-sI`  | Yes   | Very High | Low       | Completely anonymous scanning  |
| IP Protocol       | `-sO`  | Yes   | Low     | High        | Protocol enumeration           |
| SCTP INIT         | `-sY`  | Yes   | Medium  | High        | Telecom networks               |
| SCTP COOKIE-ECHO  | `-sZ`  | Yes   | High    | Medium      | Stealth SCTP                   |

*ACK scan bukan untuk mengetahui open/closed, tapi pola respons RST untuk memetakan firewall.

---

## Kombinasi Scan yang Efektif

```bash
# Kombinasi SYN + UDP (comprehensive)
sudo nmap -sS -sU -T4 -A -v 192.168.1.1

# Cepat + versi + script default
sudo nmap -sS -sV -sC --open -T4 192.168.1.1

# Paranoid dan sangat stealth
sudo nmap -sN -T1 --data-length 15 --randomize-hosts 192.168.1.0/24
```

---

*← [BAB 01: Review & Konsep Kunci](bab01.md) · Selanjutnya → [BAB 03: Advanced Target Specification & DNS Enumeration](bab03.md)*
