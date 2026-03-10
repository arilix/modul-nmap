# BAB 06 — Port Ranges & Port States

> **NMAP Guide for Beginners · 2026 Edition · Written by Rocky**

---

## Menentukan Range Port

```bash
# Scan port spesifik
nmap -p 80 192.168.1.1

# Scan beberapa port
nmap -p 22,80,443,3306 192.168.1.1

# Scan range port
nmap -p 1-1000 192.168.1.1

# Scan semua 65535 port
nmap -p- 192.168.1.1

# Scan 100 port paling umum
nmap --top-ports 100 192.168.1.1

# Scan 1000 port teratas (default)
nmap 192.168.1.1
```

---

## Penjelasan State Port

| State            | Arti                                                                                  |
|------------------|---------------------------------------------------------------------------------------|
| `open`           | Sebuah layanan aktif menerima koneksi di port ini                                     |
| `closed`         | Port dapat diakses tapi tidak ada aplikasi yang mendengarkan                          |
| `filtered`       | Firewall/filter memblokir probe Nmap; state tidak diketahui                           |
| `unfiltered`     | Port dapat diakses tapi Nmap tidak bisa menentukan open/closed                        |
| `open\|filtered` | Tidak bisa ditentukan apakah open atau filtered (UDP, FIN, dll.)                     |
| `closed\|filtered` | Tidak bisa ditentukan apakah closed atau filtered                                  |

---

> ⚠️ **Catatan:** Port yang *filtered* paling berbahaya dari sudut pandang keamanan — ini mengindikasikan aturan firewall yang mungkin menyembunyikan layanan aktif.

---

*← [BAB 05: Port Scanning Techniques](bab05.md) · Selanjutnya → [BAB 07: Service & Version Detection](bab07.md)*
