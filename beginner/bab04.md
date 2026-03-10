# BAB 04 — Host Discovery

> **NMAP Guide for Beginners · 2026 Edition · Written by Rocky**

---

Sebelum memindai port, Nmap terlebih dahulu memeriksa apakah host target sedang online. Proses ini disebut **host discovery**. Secara default, Nmap menggunakan beberapa teknik untuk menentukan apakah host aktif.

---

## Ping Scan

```bash
# Ping scan saja – tanpa port scan
nmap -sn 192.168.1.0/24

# ICMP Echo ping
nmap -PE 192.168.1.1

# TCP SYN ping ke port 443
nmap -PS443 192.168.1.1

# UDP ping
nmap -PU53 192.168.1.1
```

---

## ARP Scanning

Saat memindai jaringan lokal (LAN), Nmap secara otomatis menggunakan ARP request untuk menemukan host karena ARP lebih andal daripada ICMP di jaringan lokal dan tidak bergantung pada aturan firewall.

```bash
# Paksa ARP scan di jaringan lokal
nmap -PR 192.168.1.0/24

# ARP scan dengan output verbose
nmap -PR -v 192.168.1.0/24
```

---

## Menonaktifkan Ping (Treat All Hosts as Online)

```bash
# Lewati host discovery – scan meskipun host tidak merespons ping
nmap -Pn 192.168.1.1

# Berguna untuk host yang memblokir ICMP tapi memiliki port terbuka
```

---

## Tabel Referensi Flag Host Discovery

| Flag  | Metode Discovery       | Use Case                      |
|-------|------------------------|-------------------------------|
| `-sn` | Ping Scan (no port scan) | Quick host enumeration      |
| `-Pn` | No ping (skip discovery) | Host yang diblokir firewall |
| `-PE` | ICMP Echo              | Standard ping                 |
| `-PS` | TCP SYN ping           | Ketika ICMP diblokir          |
| `-PA` | TCP ACK ping           | Alternatif dari SYN           |
| `-PR` | ARP ping               | Scanning jaringan lokal       |

---

*← [BAB 03: Basics & Syntax](bab03.md) · Selanjutnya → [BAB 05: Port Scanning Techniques](bab05.md)*
