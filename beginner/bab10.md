# BAB 10 — Output & Reporting

> **NMAP Guide for Beginners · 2026 Edition · Written by Rocky**

---

Nmap mendukung beberapa format output, sehingga mudah untuk menyimpan hasil scan untuk analisis lebih lanjut, pelaporan, atau diproses oleh tool lain.

---

## Format Output

| Format        | Flag              | Deskripsi                                              |
|---------------|-------------------|--------------------------------------------------------|
| Normal        | `-oN scan.txt`    | Output teks yang mudah dibaca manusia                  |
| XML           | `-oX scan.xml`    | XML yang bisa di-parse mesin; bisa diimport ke tools   |
| Grepable      | `-oG scan.gnmap`  | Mudah di-parse dengan grep/awk                         |
| Script Kiddie | `-oS scan.txt`    | Format l33t-sp34k (seru, tapi tidak berguna!)          |
| Semua format  | `-oA scan`        | Simpan dalam semua format sekaligus                    |

---

## Menyimpan Hasil Scan

```bash
# Simpan sebagai teks normal
nmap -sV -oN results.txt 192.168.1.0/24

# Simpan sebagai XML untuk import ke Metasploit
nmap -sV -oX results.xml 192.168.1.0/24

# Simpan semua format sekaligus
nmap -sV -oA full_scan 192.168.1.0/24
# Membuat: full_scan.nmap, full_scan.xml, full_scan.gnmap

# Tambahkan ke file yang sudah ada
nmap -sV --append-output -oN results.txt 192.168.1.5

# Lihat hasil XML dengan browser atau parse dengan Python/Ruby
```

---

> 💡 **Tips:** Selalu simpan hasil scan dengan `-oA` saat melakukan audit resmi. Ini memberimu fleksibilitas untuk menganalisis data dalam berbagai format nanti.

---

*← [BAB 09: NSE Scripting](bab09.md) · Selanjutnya → [BAB 11: Timing & Performance](bab11.md)*
