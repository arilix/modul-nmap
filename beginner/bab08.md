# BAB 08 — OS Detection

> **NMAP Guide for Beginners · 2026 Edition · Written by Rocky**

---

Nmap dapat mengidentifikasi sistem operasi host jarak jauh dengan menganalisis perbedaan halus dalam perilaku TCP/IP stack — teknik yang dikenal sebagai **TCP/IP fingerprinting**. OS yang berbeda merespons sedikit berbeda terhadap probe jaringan yang sama.

---

## OS Fingerprinting

```bash
# Aktifkan OS detection
sudo nmap -O 192.168.1.1

# OS detection dengan version scanning
sudo nmap -O -sV 192.168.1.1

# Batasi percobaan OS detection (berguna untuk scan subnet besar)
sudo nmap -O --osscan-limit 192.168.1.0/24

# Buat tebakan OS meskipun tidak yakin
sudo nmap -O --osscan-guess 192.168.1.1
```

---

## Aggressive Detection Mode

Flag `-A` mengaktifkan OS detection, version scanning, script scanning, dan traceroute sekaligus. Ini adalah flag tunggal paling informatif di Nmap.

```bash
# Aggressive scan (OS + Version + Scripts + Traceroute)
sudo nmap -A 192.168.1.1

# Aggressive scan dengan output verbose
sudo nmap -A -v 192.168.1.1
```

**Contoh potongan output:**
```
OS details: Linux 5.15 (Ubuntu 22.04)
Network Distance: 1 hop
```

---

> 💡 **Tips:** Flag `-A` sangat berguna untuk reconnaissance awal karena memberikan gambaran lengkap tentang target dalam satu perintah.

---

*← [BAB 07: Service & Version Detection](bab07.md) · Selanjutnya → [BAB 09: Nmap Scripting Engine (NSE)](bab09.md)*
