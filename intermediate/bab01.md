# BAB 01 — Review Konsep Kunci & Memahami TCP/IP Lebih Dalam

> **NMAP Intermediate Guide · 2026 Edition**
>
> Prasyarat: Sudah memahami materi di modul Beginner (host discovery, port scanning dasar, NSE, output format).

---

## Kenapa Level Intermediate Ini Berbeda?

Di level beginner kamu belajar **apa yang dilakukan Nmap**. Di level intermediate kamu akan belajar **bagaimana dan mengapa Nmap bekerja seperti itu**, dan bagaimana memanfaatkannya secara maksimal dalam skenario nyata — termasuk jaringan yang diproteksi, automasi, dan integrasi dengan tools lain.

---

## Review Cepat: TCP/IP & Layer Jaringan

Untuk memahami teknik scanning lanjutan, kamu harus paham bagaimana paket bergerak.

```
[ Application Layer ]  HTTP, SSH, FTP, DNS
[ Transport Layer   ]  TCP, UDP, SCTP
[ Network Layer     ]  IP, ICMP
[ Data Link Layer   ]  Ethernet, ARP
[ Physical Layer    ]  Kabel, Wi-Fi
```

Nmap bekerja terutama di **layer 3 (Network) dan layer 4 (Transport)** dengan menggunakan raw sockets untuk memanipulasi paket secara langsung.

---

## TCP Three-Way Handshake (Review)

```
Client          Server
  |--- SYN ------->|   (port open: server balas SYN-ACK)
  |<-- SYN-ACK ----|
  |--- ACK ------->|

  |--- SYN ------->|   (port closed: server balas RST)
  |<-- RST --------|

  |--- SYN ------->|   (port filtered: tidak ada balasan / ICMP unreachable)
     [timeout]
```

### Mengapa SYN Scan Lebih "Stealth"?

SYN scan (`-sS`) mengirimkan SYN dan **tidak menyelesaikan handshake**. Karena koneksi tidak pernah terbentuk sepenuhnya, banyak aplikasi logging tidak mencatatnya — hanya stateful firewall / IDS yang menangkapnya.

---

## Perbedaan Port States: Detail Lebih Dalam

| State              | Trigger                                         | Implikasi                                     |
|--------------------|-------------------------------------------------|-----------------------------------------------|
| `open`             | SYN-ACK diterima (TCP) / data diterima (UDP)    | Layanan aktif, target utama recon             |
| `closed`           | RST diterima                                    | Host online, tidak ada layanan di port ini    |
| `filtered`         | Timeout / ICMP unreachable type 1,2,9,10,13     | Ada firewall; kita tidak tahu state aslinya   |
| `unfiltered`       | RST dari ACK scan                               | Port accessible tapi state tidak jelas        |
| `open\|filtered`   | Timeout pada UDP/Null/FIN/Xmas scan             | Nmap tidak bisa membedakan                    |
| `closed\|filtered` | Hanya pada Idle scan                            | IP ID tidak berubah                           |

---

## Flags TCP yang Digunakan Nmap

| Flag | Bit  | Fungsi                                        |
|------|------|-----------------------------------------------|
| SYN  | 0x02 | Inisiasi koneksi                              |
| ACK  | 0x10 | Konfirmasi penerimaan                         |
| RST  | 0x04 | Reset/tutup koneksi secara paksa              |
| FIN  | 0x01 | Terminasi koneksi secara normal               |
| PSH  | 0x08 | Push data segera ke aplikasi                  |
| URG  | 0x20 | Data urgent                                   |

> 💡 **Xmas scan** (`-sX`) menyalakan flag FIN + PSH + URG sekaligus — seperti "lampu natal" yang menyala semua.

---

## Konsep Raw Sockets & Mengapa Butuh Root

Nmap menggunakan **raw sockets** untuk membangun paket dari nol (termasuk header IP dan TCP). Ini membutuhkan hak akses root/Administrator karena:
- OS normal tidak mengizinkan aplikasi user space memanipulasi header network secara langsung
- Npcap (Windows) / libpcap (Linux/macOS) menjadi jembatan antara Nmap dan driver jaringan

```bash
# Tanpa root: hanya TCP Connect scan yang bisa dilakukan
nmap -sT 192.168.1.1

# Dengan root: semua jenis scan tersedia
sudo nmap -sS 192.168.1.1
```

---

## Nmap Probe Database

Nmap menyimpan database probe untuk version detection di:
```
/usr/share/nmap/nmap-service-probes   # Probe untuk version detection
/usr/share/nmap/nmap-services         # Database nama layanan & port
/usr/share/nmap/nmap-os-db            # Database OS fingerprint
/usr/share/nmap/scripts/              # NSE scripts
```

Memahami file-file ini akan sangat berguna saat kamu melakukan tuning atau debugging scan.

---

## Latihan

```bash
# Lihat database layanan Nmap
cat /usr/share/nmap/nmap-services | grep "^http\b"

# Lihat daftar semua NSE scripts
ls /usr/share/nmap/scripts/ | wc -l

# Cek versi Nmap dan fitur yang dikompilasi
nmap --version
```

---

*Selanjutnya → [BAB 02: Advanced Scan Techniques](bab02.md)*
