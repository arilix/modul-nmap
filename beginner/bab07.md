# BAB 07 — Service & Version Detection

> **NMAP Guide for Beginners · 2026 Edition · Written by Rocky**

---

Mengetahui bahwa port 80 terbuka itu berguna, tetapi mengetahui bahwa port itu menjalankan **Apache 2.4.52 di Ubuntu 22.04** jauh lebih berharga. Fitur version detection Nmap melakukan probe pada port terbuka dan mengidentifikasi software yang berjalan di dalamnya.

---

## Version Scanning

```bash
# Aktifkan version detection
nmap -sV 192.168.1.1

# Version detection agresif
nmap -sV --version-intensity 9 192.168.1.1

# Version detection dengan probing ringan
nmap -sV --version-intensity 2 192.168.1.1
```

---

## Level Intensitas

Version intensity berkisar dari **0** (paling ringan, paling cepat) hingga **9** (paling berat, paling akurat). Default-nya adalah **7**.

| Intensitas | Flag                    | Deskripsi                                |
|------------|-------------------------|------------------------------------------|
| 0          | `--version-intensity 0` | Sangat ringan, hanya probe paling umum   |
| 2          | `--version-light`       | Probe ringan, traffic jaringan lebih kecil |
| 7          | *(default)*             | Keseimbangan antara akurasi dan kecepatan |
| 9          | `--version-intensity 9` | Paling agresif, akurasi maksimum         |
| All probes | `--version-all`         | Coba setiap probe yang ada               |

---

> 💡 **Catatan:** Level intensitas yang lebih tinggi meningkatkan akurasi tetapi juga meningkatkan waktu scan dan kebisingan jaringan. Gunakan nilai tinggi hanya ketika presisi benar-benar penting.

---

*← [BAB 06: Port Ranges & States](bab06.md) · Selanjutnya → [BAB 08: OS Detection](bab08.md)*
