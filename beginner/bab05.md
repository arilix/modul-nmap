# BAB 05 — Port Scanning Techniques

> **NMAP Guide for Beginners · 2026 Edition · Written by Rocky**

---

Nmap menawarkan berbagai teknik scanning, masing-masing cocok untuk skenario yang berbeda. Memilih jenis scan yang tepat tergantung pada tujuanmu, kondisi jaringan, dan level izin yang kamu miliki.

---

## TCP SYN Scan (Stealth Scan)

Jenis scan paling default dan populer. Disebut *stealth scan* karena tidak menyelesaikan TCP handshake, sehingga lebih sulit dicatat oleh firewall sederhana.

```bash
# SYN scan (membutuhkan hak root/admin)
sudo nmap -sS 192.168.1.1

# Cara kerjanya:
# Client kirim SYN → Server balas SYN-ACK (open) atau RST (closed)
# Client kirim RST → Koneksi tidak pernah sepenuhnya terbentuk
```

---

## TCP Connect Scan

Digunakan saat kamu tidak punya hak raw packet. Scan ini menyelesaikan full three-way handshake, sehingga lebih mudah terdeteksi tapi berjalan tanpa hak akses elevated.

```bash
# TCP Connect scan (tidak perlu root)
nmap -sT 192.168.1.1
```

---

## UDP Scan

Banyak layanan penting menggunakan UDP (DNS, SNMP, DHCP). UDP scanning lebih lambat dari TCP karena tidak ada handshake untuk konfirmasi port terbuka.

```bash
# UDP scan
sudo nmap -sU 192.168.1.1

# Kombinasi UDP dan TCP
sudo nmap -sS -sU 192.168.1.1
```

---

## Jenis Scan Lainnya

| Scan Type   | Flag   | Deskripsi                                      |
|-------------|--------|------------------------------------------------|
| Null Scan   | `-sN`  | Tidak ada flag; port tertutup membalas RST     |
| FIN Scan    | `-sF`  | Hanya flag FIN; menghindari beberapa firewall  |
| Xmas Scan   | `-sX`  | Flag FIN+PSH+URG; mencolok namun berguna       |
| ACK Scan    | `-sA`  | Memetakan aturan firewall                      |
| Window Scan | `-sW`  | Mirip ACK; mengeksploitasi TCP window          |
| Maimon Scan | `-sM`  | Probe FIN/ACK; teknik lama                     |
| SCTP INIT   | `-sY`  | Untuk layanan protokol SCTP                    |

---

*← [BAB 04: Host Discovery](bab04.md) · Selanjutnya → [BAB 06: Port Ranges & Port States](bab06.md)*
