# BAB 12 — Firewall & IDS Evasion

> **NMAP Guide for Beginners · 2026 Edition · Written by Rocky**

---

Saat melakukan penetration test yang diizinkan, mungkin kamu perlu menghindari firewall atau intrusion detection system. Nmap menyediakan beberapa teknik untuk keperluan ini.

---

## Packet Fragmentation

Memecah paket menjadi fragmen kecil dapat membingungkan beberapa firewall dan packet filter sederhana.

```bash
# Fragment paket menjadi potongan 8-byte
sudo nmap -f 192.168.1.1

# Fragment menjadi potongan 16-byte
sudo nmap -ff 192.168.1.1

# Tentukan ukuran fragment secara manual
sudo nmap --mtu 16 192.168.1.1
```

---

## Decoy Scanning

Buat scan-mu seolah-olah berasal dari beberapa alamat IP secara bersamaan, menyembunyikan sumber aslimu dalam kebisingan.

```bash
# Gunakan IP decoy (ME = posisi IP aslimu)
sudo nmap -D 10.0.0.1,10.0.0.2,ME 192.168.1.1

# Generate decoy secara acak
sudo nmap -D RND:10 192.168.1.1
```

---

## Source IP Spoofing & Port

```bash
# Spoof source IP (kamu tidak akan menerima balasan)
sudo nmap -S 1.2.3.4 192.168.1.1

# Gunakan source port spesifik (misal: 53 = DNS, sering di-whitelist)
sudo nmap --source-port 53 192.168.1.1
sudo nmap -g 80 192.168.1.1

# Spoof MAC address
sudo nmap --spoof-mac 0 192.168.1.1        # MAC acak
sudo nmap --spoof-mac Cisco 192.168.1.1    # MAC vendor tertentu
```

---

> ⚠️ **Catatan Penting:** Semua teknik evasion hanya boleh digunakan selama penetration testing yang diizinkan atau di jaringan yang kamu miliki. Penggunaan tanpa izin adalah ilegal.

---

*← [BAB 11: Timing & Performance](bab11.md) · Selanjutnya → [BAB 13: Zenmap GUI](bab13.md)*
