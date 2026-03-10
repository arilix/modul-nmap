# BAB 15 — Nmap Cheat Sheet & Penutup

> **NMAP Guide for Beginners · 2026 Edition · Written by Rocky**

---

Referensi cepat perintah-perintah Nmap yang paling sering digunakan sehari-hari.

---

## Jenis-Jenis Scan

| Perintah              | Deskripsi                      |
|-----------------------|-------------------------------|
| `nmap -sS <target>`   | TCP SYN Scan (Stealth)         |
| `nmap -sT <target>`   | TCP Connect Scan               |
| `nmap -sU <target>`   | UDP Scan                       |
| `nmap -sN <target>`   | Null Scan                      |
| `nmap -sF <target>`   | FIN Scan                       |
| `nmap -sX <target>`   | Xmas Scan                      |
| `nmap -sA <target>`   | ACK Scan (pemetaan firewall)   |
| `nmap -sn <target>`   | Ping Scan (tanpa port)         |

---

## Discovery & Port

| Perintah                   | Deskripsi                        |
|----------------------------|----------------------------------|
| `nmap -Pn <target>`        | Skip ping, anggap host online    |
| `nmap -p 80 <target>`      | Scan port 80                     |
| `nmap -p 1-1000 <target>`  | Scan range port                  |
| `nmap -p- <target>`        | Scan semua 65535 port            |
| `nmap --top-ports 100 <target>` | Scan 100 port teratas       |
| `nmap --open <target>`     | Tampilkan hanya port terbuka     |

---

## Detection & Info

| Perintah                         | Deskripsi                              |
|----------------------------------|----------------------------------------|
| `nmap -sV <target>`              | Version detection                      |
| `nmap -O <target>`               | OS detection                           |
| `nmap -A <target>`               | Agresif (OS+Ver+Script+Traceroute)     |
| `nmap -sC <target>`              | Scan default script                    |
| `nmap --script=vuln <target>`    | Script vulnerability                   |
| `nmap --script=<name> <target>`  | Jalankan NSE script spesifik           |

---

## Output & Timing

| Perintah                         | Deskripsi                   |
|----------------------------------|-----------------------------|
| `nmap -oN file.txt <target>`     | Simpan output normal        |
| `nmap -oX file.xml <target>`     | Simpan output XML           |
| `nmap -oA scan <target>`         | Simpan semua format         |
| `nmap -v <target>`               | Verbose output              |
| `nmap -vv <target>`              | Very verbose                |
| `nmap -T4 <target>`              | Timing agresif              |
| `nmap -T2 <target>`              | Timing polite               |

---

## Evasion & Misc

| Perintah                          | Deskripsi                     |
|-----------------------------------|-------------------------------|
| `nmap -f <target>`                | Fragment paket                |
| `nmap -D RND:10 <target>`         | Random decoys                 |
| `nmap -g 53 <target>`             | Gunakan port 53 sebagai source |
| `nmap --spoof-mac 0 <target>`     | MAC address acak              |
| `nmap -iL hosts.txt`              | Input dari file               |
| `nmap --exclude 192.168.1.5 <target>` | Exclude satu host         |
| `nmap --reason <target>`          | Tampilkan alasan state port   |
| `nmap --traceroute <target>`      | Sertakan traceroute           |

---

## Penutup

Kamu sekarang telah mempelajari fondasi penting Nmap — mulai dari host discovery dan port scanning dasar hingga NSE scripting lanjutan dan teknik evasion. Nmap tetap menjadi salah satu tool paling serbaguna dan banyak digunakan dalam toolkit profesional keamanan manapun.

**Lanjutkan perjalanan belajarmu dengan:**
- Berlatih di sistem yang sengaja dibuat rentan seperti **Metasploitable**, **HackTheBox**, atau **TryHackMe**
- Jelajahi dokumentasi Nmap lengkap di **nmap.org/book**
- Tulis custom NSE scripts dalam bahasa Lua
- Kombinasikan output Nmap dengan tools seperti Metasploit, OpenVAS, dan Nessus
- Bangun home lab untuk latihan ethical hacking

---

> **PENTING — PERINGATAN HUKUM**
>
> Memindai jaringan, sistem, atau perangkat tanpa otorisasi tertulis dari pemiliknya adalah **ilegal** di sebagian besar negara dan dapat mengakibatkan tuntutan pidana. Selalu dapatkan izin tertulis sebelum memindai jaringan apapun. Nmap dirancang untuk administrasi jaringan yang diizinkan dan audit keamanan. Gunakan secara bertanggung jawab dan etis setiap saat.

---

*Happy Scanning! 🎯*

*Written by Rocky · 2026 Edition · www.codelivly.com*

---

*← [BAB 14: Real-World Use Cases](bab14.md)*
